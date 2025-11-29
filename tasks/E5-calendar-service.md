# E5: Atlas.Calendar - Unified Calendar Service

## Overview

Centralized calendar abstraction providing market holidays, trading hours, and economic event scheduling across all ATLAS collectors. Implements Quartz.NET `ICalendar` for holiday-aware job scheduling.

**Status**: Planning  
**Priority**: P1 (enables FinnhubCollector market-aware polling)  
**Estimated Effort**: 8-10 hours  
**Dependencies**: Nager.Date NuGet, Finnhub API (optional Phase 2)

---

## Problem Statement

Calendar logic is scattered across collectors:
- FredCollector: Ad-hoc holiday skipping in Quartz triggers
- FinnhubCollector: Separate market hours logic via `/market-status`
- ThresholdEngine: No awareness of data release schedules

Need single source of truth for:
1. US market holidays (NYSE/NASDAQ closures)
2. Trading session times (regular, early close, extended hours)
3. Economic event calendar (Fed meetings, CPI releases)
4. FRED release schedules per series

---

## Architecture

```
Atlas.Calendar/
├── Atlas.Calendar.csproj
├── Models/
│   ├── MarketHoliday.cs
│   ├── TradingSession.cs
│   ├── EconomicEvent.cs
│   └── Enums/
│       ├── MarketStatus.cs
│       └── EconomicEventType.cs
├── Abstractions/
│   ├── IMarketCalendar.cs
│   ├── IEconomicCalendar.cs
│   └── ITradingHoursProvider.cs
├── Providers/
│   ├── NyseMarketCalendar.cs         # Primary implementation
│   ├── NagerHolidayProvider.cs       # Nager.Date integration
│   ├── FinnhubEconomicCalendar.cs    # Phase 2
│   └── StaticNyseHolidays.cs         # NYSE-specific overrides
├── Quartz/
│   ├── MarketHolidayCalendar.cs      # ICalendar implementation
│   └── TradingHoursCalendar.cs       # Skip non-market hours
└── Extensions/
    └── ServiceCollectionExtensions.cs
```

---

## Models

### MarketHoliday.cs

```csharp
namespace Atlas.Calendar.Models;

/// <summary>
/// Represents a market closure or early close day.
/// </summary>
public sealed record MarketHoliday
{
    public required DateOnly Date { get; init; }
    public required string Name { get; init; }
    public required MarketHolidayType Type { get; init; }
    
    /// <summary>
    /// For early close days, the close time in ET.
    /// Null for full closures.
    /// </summary>
    public TimeOnly? EarlyCloseTime { get; init; }
    
    /// <summary>
    /// Source of the holiday data (Nager, NYSE, Manual).
    /// </summary>
    public string Source { get; init; } = "NYSE";
}

public enum MarketHolidayType
{
    FullClosure,    // Market closed all day
    EarlyClose,     // Market closes early (typically 1:00 PM ET)
    Observed        // Holiday observed on different date
}
```

### TradingSession.cs

```csharp
namespace Atlas.Calendar.Models;

/// <summary>
/// Trading session times for a specific date.
/// All times in Eastern Time (ET).
/// </summary>
public sealed record TradingSession
{
    public required DateOnly Date { get; init; }
    public required MarketStatus Status { get; init; }
    
    // Pre-market: 4:00 AM - 9:30 AM ET
    public TimeOnly? PreMarketOpen { get; init; }
    public TimeOnly? PreMarketClose { get; init; }
    
    // Regular: 9:30 AM - 4:00 PM ET (or early close time)
    public TimeOnly? RegularOpen { get; init; }
    public TimeOnly? RegularClose { get; init; }
    
    // After-hours: 4:00 PM - 8:00 PM ET
    public TimeOnly? AfterHoursOpen { get; init; }
    public TimeOnly? AfterHoursClose { get; init; }
    
    /// <summary>
    /// Check if a given ET time falls within regular trading hours.
    /// </summary>
    public bool IsRegularHours(TimeOnly timeEt)
    {
        if (Status == MarketStatus.Closed) return false;
        if (RegularOpen is null || RegularClose is null) return false;
        return timeEt >= RegularOpen && timeEt < RegularClose;
    }
    
    /// <summary>
    /// Check if market is open (any session).
    /// </summary>
    public bool IsOpen(TimeOnly timeEt)
    {
        if (Status == MarketStatus.Closed) return false;
        
        // Check pre-market
        if (PreMarketOpen is not null && PreMarketClose is not null &&
            timeEt >= PreMarketOpen && timeEt < PreMarketClose)
            return true;
            
        // Check regular
        if (IsRegularHours(timeEt)) return true;
        
        // Check after-hours
        if (AfterHoursOpen is not null && AfterHoursClose is not null &&
            timeEt >= AfterHoursOpen && timeEt < AfterHoursClose)
            return true;
            
        return false;
    }
}

public enum MarketStatus
{
    Closed,
    PreMarket,
    Open,
    AfterHours
}
```

### EconomicEvent.cs

```csharp
namespace Atlas.Calendar.Models;

/// <summary>
/// Economic calendar event (Fed meetings, CPI releases, etc).
/// </summary>
public sealed record EconomicEvent
{
    public required string EventId { get; init; }
    public required string Name { get; init; }
    public required EconomicEventType Type { get; init; }
    public required DateTimeOffset EventTime { get; init; }
    public required string Country { get; init; }
    public required EconomicImpact Impact { get; init; }
    
    // Expected vs Actual (populated after release)
    public string? Estimate { get; init; }
    public string? Actual { get; init; }
    public string? Previous { get; init; }
    public string? Unit { get; init; }
    
    // Source tracking
    public string Source { get; init; } = "Finnhub";
    public DateTimeOffset CollectedAt { get; init; }
}

public enum EconomicEventType
{
    FedMeeting,         // FOMC meetings
    FedMinutes,         // FOMC minutes release
    FedSpeech,          // Fed official speeches
    CpiRelease,         // Consumer Price Index
    PpiRelease,         // Producer Price Index
    EmploymentReport,   // Nonfarm payrolls, unemployment
    GdpRelease,         // GDP reports
    RetailSales,        // Retail sales data
    IsmManufacturing,   // ISM PMI
    IsmServices,        // ISM Services PMI
    ConsumerSentiment,  // UMich sentiment
    HousingData,        // Housing starts, permits
    TradeBalance,       // Trade deficit
    Other
}

public enum EconomicImpact
{
    Low,
    Medium,
    High
}
```

---

## Abstractions

### IMarketCalendar.cs

```csharp
namespace Atlas.Calendar.Abstractions;

/// <summary>
/// Provides market holiday and trading day information.
/// Primary interface for Quartz calendar integration.
/// </summary>
public interface IMarketCalendar
{
    #region Holiday Queries
    
    /// <summary>
    /// Check if the market is fully closed on a given date.
    /// </summary>
    bool IsMarketClosed(DateOnly date);
    
    /// <summary>
    /// Check if the market has an early close on a given date.
    /// </summary>
    bool IsEarlyClose(DateOnly date);
    
    /// <summary>
    /// Get the early close time if applicable, null otherwise.
    /// </summary>
    TimeOnly? GetEarlyCloseTime(DateOnly date);
    
    /// <summary>
    /// Get all market holidays for a given year.
    /// </summary>
    IReadOnlyList<MarketHoliday> GetHolidays(int year);
    
    /// <summary>
    /// Get holiday details for a specific date, null if not a holiday.
    /// </summary>
    MarketHoliday? GetHoliday(DateOnly date);
    
    #endregion
    
    #region Trading Day Navigation
    
    /// <summary>
    /// Check if a date is a valid trading day (weekday + not holiday).
    /// </summary>
    bool IsTradingDay(DateOnly date);
    
    /// <summary>
    /// Get the next trading day after the given date.
    /// </summary>
    DateOnly GetNextTradingDay(DateOnly from);
    
    /// <summary>
    /// Get the previous trading day before the given date.
    /// </summary>
    DateOnly GetPreviousTradingDay(DateOnly from);
    
    /// <summary>
    /// Get N trading days from a starting date (positive = forward, negative = backward).
    /// </summary>
    DateOnly GetTradingDayOffset(DateOnly from, int tradingDays);
    
    /// <summary>
    /// Count trading days between two dates (exclusive of end date).
    /// </summary>
    int CountTradingDays(DateOnly from, DateOnly to);
    
    #endregion
    
    #region Real-time Status
    
    /// <summary>
    /// Get the current market status based on current time.
    /// </summary>
    MarketStatus GetCurrentStatus();
    
    /// <summary>
    /// Get the current market status for a specific datetime.
    /// </summary>
    MarketStatus GetStatus(DateTimeOffset dateTime);
    
    /// <summary>
    /// Check if the market is currently open (regular hours only).
    /// </summary>
    bool IsMarketOpenNow();
    
    #endregion
}
```

### ITradingHoursProvider.cs

```csharp
namespace Atlas.Calendar.Abstractions;

/// <summary>
/// Provides detailed trading session information.
/// </summary>
public interface ITradingHoursProvider
{
    /// <summary>
    /// Get the trading session for a specific date.
    /// </summary>
    TradingSession GetSession(DateOnly date);
    
    /// <summary>
    /// Get trading sessions for a date range.
    /// </summary>
    IReadOnlyList<TradingSession> GetSessions(DateOnly from, DateOnly to);
    
    /// <summary>
    /// Get the next market open time from a given datetime.
    /// </summary>
    DateTimeOffset GetNextMarketOpen(DateTimeOffset from);
    
    /// <summary>
    /// Get the next market close time from a given datetime.
    /// </summary>
    DateTimeOffset GetNextMarketClose(DateTimeOffset from);
    
    /// <summary>
    /// Get time until market opens (zero if currently open).
    /// </summary>
    TimeSpan GetTimeUntilOpen(DateTimeOffset from);
}
```

### IEconomicCalendar.cs

```csharp
namespace Atlas.Calendar.Abstractions;

/// <summary>
/// Provides economic event calendar data.
/// Phase 2: Backed by Finnhub /calendar/economic endpoint.
/// </summary>
public interface IEconomicCalendar
{
    /// <summary>
    /// Get upcoming economic events within a time window.
    /// </summary>
    Task<IReadOnlyList<EconomicEvent>> GetUpcomingEventsAsync(
        int daysAhead = 30,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get events filtered by type.
    /// </summary>
    Task<IReadOnlyList<EconomicEvent>> GetEventsByTypeAsync(
        EconomicEventType type,
        DateOnly from,
        DateOnly to,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get high-impact events only.
    /// </summary>
    Task<IReadOnlyList<EconomicEvent>> GetHighImpactEventsAsync(
        int daysAhead = 14,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Get all Fed-related events (meetings, minutes, speeches).
    /// </summary>
    Task<IReadOnlyList<EconomicEvent>> GetFedEventsAsync(
        int year,
        CancellationToken cancellationToken = default);
    
    /// <summary>
    /// Check if there's a high-impact event on a specific date.
    /// </summary>
    Task<bool> HasHighImpactEventAsync(
        DateOnly date,
        CancellationToken cancellationToken = default);
}
```

---

## Providers

### StaticNyseHolidays.cs

```csharp
namespace Atlas.Calendar.Providers;

/// <summary>
/// NYSE-specific holiday rules that differ from federal holidays.
/// Updated annually - review each December for next year.
/// </summary>
internal static class StaticNyseHolidays
{
    // Standard NYSE holidays (observed rules apply)
    // - New Year's Day
    // - Martin Luther King Jr. Day (3rd Monday Jan)
    // - Presidents Day (3rd Monday Feb)
    // - Good Friday (NOT a federal holiday)
    // - Memorial Day (last Monday May)
    // - Juneteenth (June 19)
    // - Independence Day (July 4)
    // - Labor Day (1st Monday Sep)
    // - Thanksgiving (4th Thursday Nov)
    // - Christmas Day
    
    /// <summary>
    /// Good Friday dates (NYSE closed, not federal holiday).
    /// Must be calculated from Easter.
    /// </summary>
    public static DateOnly GetGoodFriday(int year)
    {
        // Anonymous Gregorian algorithm for Easter Sunday
        var a = year % 19;
        var b = year / 100;
        var c = year % 100;
        var d = b / 4;
        var e = b % 4;
        var f = (b + 8) / 25;
        var g = (b - f + 1) / 3;
        var h = (19 * a + b - d - g + 15) % 30;
        var i = c / 4;
        var k = c % 4;
        var l = (32 + 2 * e + 2 * i - h - k) % 7;
        var m = (a + 11 * h + 22 * l) / 451;
        var month = (h + l - 7 * m + 114) / 31;
        var day = ((h + l - 7 * m + 114) % 31) + 1;
        
        var easter = new DateOnly(year, month, day);
        return easter.AddDays(-2); // Good Friday is 2 days before Easter
    }
    
    /// <summary>
    /// Early close days (1:00 PM ET close).
    /// Typically: Day before Independence Day, Black Friday, Christmas Eve.
    /// </summary>
    public static IReadOnlyList<MarketHoliday> GetEarlyCloses(int year)
    {
        var earlyCloses = new List<MarketHoliday>();
        var earlyCloseTime = new TimeOnly(13, 0); // 1:00 PM ET
        
        // Day before Independence Day (if July 4 is not Monday)
        var july4 = new DateOnly(year, 7, 4);
        if (july4.DayOfWeek != DayOfWeek.Monday)
        {
            var dayBefore = july4.DayOfWeek == DayOfWeek.Sunday 
                ? july4.AddDays(-2)  // Friday before
                : july4.AddDays(-1); // Day before
            
            earlyCloses.Add(new MarketHoliday
            {
                Date = dayBefore,
                Name = "Day Before Independence Day",
                Type = MarketHolidayType.EarlyClose,
                EarlyCloseTime = earlyCloseTime,
                Source = "NYSE"
            });
        }
        
        // Black Friday (day after Thanksgiving)
        var thanksgiving = GetNthWeekdayOfMonth(year, 11, DayOfWeek.Thursday, 4);
        earlyCloses.Add(new MarketHoliday
        {
            Date = thanksgiving.AddDays(1),
            Name = "Day After Thanksgiving",
            Type = MarketHolidayType.EarlyClose,
            EarlyCloseTime = earlyCloseTime,
            Source = "NYSE"
        });
        
        // Christmas Eve (if not weekend)
        var christmasEve = new DateOnly(year, 12, 24);
        if (christmasEve.DayOfWeek is not (DayOfWeek.Saturday or DayOfWeek.Sunday))
        {
            earlyCloses.Add(new MarketHoliday
            {
                Date = christmasEve,
                Name = "Christmas Eve",
                Type = MarketHolidayType.EarlyClose,
                EarlyCloseTime = earlyCloseTime,
                Source = "NYSE"
            });
        }
        
        return earlyCloses;
    }
    
    /// <summary>
    /// Get full closure holidays for a year.
    /// </summary>
    public static IReadOnlyList<MarketHoliday> GetFullClosures(int year)
    {
        var closures = new List<MarketHoliday>
        {
            // New Year's Day (observed)
            ObservedHoliday(new DateOnly(year, 1, 1), "New Year's Day"),
            
            // MLK Day (3rd Monday January)
            new MarketHoliday
            {
                Date = GetNthWeekdayOfMonth(year, 1, DayOfWeek.Monday, 3),
                Name = "Martin Luther King Jr. Day",
                Type = MarketHolidayType.FullClosure,
                Source = "NYSE"
            },
            
            // Presidents Day (3rd Monday February)
            new MarketHoliday
            {
                Date = GetNthWeekdayOfMonth(year, 2, DayOfWeek.Monday, 3),
                Name = "Presidents Day",
                Type = MarketHolidayType.FullClosure,
                Source = "NYSE"
            },
            
            // Good Friday
            new MarketHoliday
            {
                Date = GetGoodFriday(year),
                Name = "Good Friday",
                Type = MarketHolidayType.FullClosure,
                Source = "NYSE"
            },
            
            // Memorial Day (last Monday May)
            new MarketHoliday
            {
                Date = GetLastWeekdayOfMonth(year, 5, DayOfWeek.Monday),
                Name = "Memorial Day",
                Type = MarketHolidayType.FullClosure,
                Source = "NYSE"
            },
            
            // Juneteenth (observed)
            ObservedHoliday(new DateOnly(year, 6, 19), "Juneteenth"),
            
            // Independence Day (observed)
            ObservedHoliday(new DateOnly(year, 7, 4), "Independence Day"),
            
            // Labor Day (1st Monday September)
            new MarketHoliday
            {
                Date = GetNthWeekdayOfMonth(year, 9, DayOfWeek.Monday, 1),
                Name = "Labor Day",
                Type = MarketHolidayType.FullClosure,
                Source = "NYSE"
            },
            
            // Thanksgiving (4th Thursday November)
            new MarketHoliday
            {
                Date = GetNthWeekdayOfMonth(year, 11, DayOfWeek.Thursday, 4),
                Name = "Thanksgiving",
                Type = MarketHolidayType.FullClosure,
                Source = "NYSE"
            },
            
            // Christmas (observed)
            ObservedHoliday(new DateOnly(year, 12, 25), "Christmas Day")
        };
        
        return closures;
    }
    
    private static MarketHoliday ObservedHoliday(DateOnly actualDate, string name)
    {
        var observedDate = actualDate.DayOfWeek switch
        {
            DayOfWeek.Saturday => actualDate.AddDays(-1), // Friday
            DayOfWeek.Sunday => actualDate.AddDays(1),    // Monday
            _ => actualDate
        };
        
        return new MarketHoliday
        {
            Date = observedDate,
            Name = name,
            Type = observedDate == actualDate 
                ? MarketHolidayType.FullClosure 
                : MarketHolidayType.Observed,
            Source = "NYSE"
        };
    }
    
    private static DateOnly GetNthWeekdayOfMonth(int year, int month, DayOfWeek dayOfWeek, int n)
    {
        var first = new DateOnly(year, month, 1);
        var daysUntil = ((int)dayOfWeek - (int)first.DayOfWeek + 7) % 7;
        return first.AddDays(daysUntil + (n - 1) * 7);
    }
    
    private static DateOnly GetLastWeekdayOfMonth(int year, int month, DayOfWeek dayOfWeek)
    {
        var last = new DateOnly(year, month, DateTime.DaysInMonth(year, month));
        var daysSince = ((int)last.DayOfWeek - (int)dayOfWeek + 7) % 7;
        return last.AddDays(-daysSince);
    }
}
```

### NyseMarketCalendar.cs

```csharp
using System.Collections.Concurrent;
using Microsoft.Extensions.Logging;

namespace Atlas.Calendar.Providers;

/// <summary>
/// Primary implementation of IMarketCalendar for NYSE/NASDAQ.
/// </summary>
public sealed class NyseMarketCalendar : IMarketCalendar, ITradingHoursProvider
{
    private readonly ILogger<NyseMarketCalendar> _logger;
    private readonly TimeProvider _timeProvider;
    private readonly ConcurrentDictionary<int, IReadOnlyList<MarketHoliday>> _holidayCache = new();
    
    // Eastern Time zone for NYSE
    private static readonly TimeZoneInfo EasternTime = 
        TimeZoneInfo.FindSystemTimeZoneById("America/New_York");
    
    // Standard trading hours (ET)
    private static readonly TimeOnly PreMarketOpen = new(4, 0);
    private static readonly TimeOnly RegularOpen = new(9, 30);
    private static readonly TimeOnly RegularClose = new(16, 0);
    private static readonly TimeOnly AfterHoursClose = new(20, 0);
    private static readonly TimeOnly EarlyClose = new(13, 0);
    
    public NyseMarketCalendar(
        ILogger<NyseMarketCalendar> logger,
        TimeProvider? timeProvider = null)
    {
        _logger = logger;
        _timeProvider = timeProvider ?? TimeProvider.System;
    }
    
    #region IMarketCalendar Implementation
    
    public bool IsMarketClosed(DateOnly date)
    {
        // Weekend check
        if (date.DayOfWeek is DayOfWeek.Saturday or DayOfWeek.Sunday)
            return true;
        
        var holiday = GetHoliday(date);
        return holiday?.Type == MarketHolidayType.FullClosure ||
               holiday?.Type == MarketHolidayType.Observed;
    }
    
    public bool IsEarlyClose(DateOnly date)
    {
        var holiday = GetHoliday(date);
        return holiday?.Type == MarketHolidayType.EarlyClose;
    }
    
    public TimeOnly? GetEarlyCloseTime(DateOnly date)
    {
        var holiday = GetHoliday(date);
        return holiday?.EarlyCloseTime;
    }
    
    public IReadOnlyList<MarketHoliday> GetHolidays(int year)
    {
        return _holidayCache.GetOrAdd(year, y =>
        {
            var holidays = new List<MarketHoliday>();
            holidays.AddRange(StaticNyseHolidays.GetFullClosures(y));
            holidays.AddRange(StaticNyseHolidays.GetEarlyCloses(y));
            
            _logger.LogDebug("Loaded {Count} holidays for year {Year}", holidays.Count, y);
            return holidays.OrderBy(h => h.Date).ToList();
        });
    }
    
    public MarketHoliday? GetHoliday(DateOnly date)
    {
        var holidays = GetHolidays(date.Year);
        return holidays.FirstOrDefault(h => h.Date == date);
    }
    
    public bool IsTradingDay(DateOnly date)
    {
        return !IsMarketClosed(date);
    }
    
    public DateOnly GetNextTradingDay(DateOnly from)
    {
        var next = from.AddDays(1);
        while (!IsTradingDay(next))
        {
            next = next.AddDays(1);
            
            // Safety: don't loop forever
            if (next > from.AddDays(14))
            {
                _logger.LogWarning("Could not find trading day within 14 days of {Date}", from);
                throw new InvalidOperationException($"No trading day found within 14 days of {from}");
            }
        }
        return next;
    }
    
    public DateOnly GetPreviousTradingDay(DateOnly from)
    {
        var prev = from.AddDays(-1);
        while (!IsTradingDay(prev))
        {
            prev = prev.AddDays(-1);
            
            if (prev < from.AddDays(-14))
            {
                _logger.LogWarning("Could not find trading day within 14 days before {Date}", from);
                throw new InvalidOperationException($"No trading day found within 14 days before {from}");
            }
        }
        return prev;
    }
    
    public DateOnly GetTradingDayOffset(DateOnly from, int tradingDays)
    {
        var current = from;
        var direction = tradingDays > 0 ? 1 : -1;
        var remaining = Math.Abs(tradingDays);
        
        while (remaining > 0)
        {
            current = current.AddDays(direction);
            if (IsTradingDay(current))
                remaining--;
        }
        
        return current;
    }
    
    public int CountTradingDays(DateOnly from, DateOnly to)
    {
        if (to <= from) return 0;
        
        var count = 0;
        var current = from;
        while (current < to)
        {
            current = current.AddDays(1);
            if (IsTradingDay(current))
                count++;
        }
        return count;
    }
    
    public MarketStatus GetCurrentStatus()
    {
        return GetStatus(_timeProvider.GetUtcNow());
    }
    
    public MarketStatus GetStatus(DateTimeOffset dateTime)
    {
        var eastern = TimeZoneInfo.ConvertTime(dateTime, EasternTime);
        var date = DateOnly.FromDateTime(eastern.DateTime);
        var time = TimeOnly.FromDateTime(eastern.DateTime);
        
        if (IsMarketClosed(date))
            return MarketStatus.Closed;
        
        var closeTime = GetEarlyCloseTime(date) ?? RegularClose;
        
        if (time < PreMarketOpen)
            return MarketStatus.Closed;
        if (time < RegularOpen)
            return MarketStatus.PreMarket;
        if (time < closeTime)
            return MarketStatus.Open;
        if (time < AfterHoursClose && !IsEarlyClose(date))
            return MarketStatus.AfterHours;
        
        return MarketStatus.Closed;
    }
    
    public bool IsMarketOpenNow()
    {
        return GetCurrentStatus() == MarketStatus.Open;
    }
    
    #endregion
    
    #region ITradingHoursProvider Implementation
    
    public TradingSession GetSession(DateOnly date)
    {
        if (IsMarketClosed(date))
        {
            return new TradingSession
            {
                Date = date,
                Status = MarketStatus.Closed
            };
        }
        
        var isEarly = IsEarlyClose(date);
        var closeTime = GetEarlyCloseTime(date) ?? RegularClose;
        
        return new TradingSession
        {
            Date = date,
            Status = MarketStatus.Closed, // Will be recalculated by caller based on time
            PreMarketOpen = PreMarketOpen,
            PreMarketClose = RegularOpen,
            RegularOpen = RegularOpen,
            RegularClose = closeTime,
            AfterHoursOpen = isEarly ? null : closeTime,
            AfterHoursClose = isEarly ? null : AfterHoursClose
        };
    }
    
    public IReadOnlyList<TradingSession> GetSessions(DateOnly from, DateOnly to)
    {
        var sessions = new List<TradingSession>();
        var current = from;
        
        while (current <= to)
        {
            sessions.Add(GetSession(current));
            current = current.AddDays(1);
        }
        
        return sessions;
    }
    
    public DateTimeOffset GetNextMarketOpen(DateTimeOffset from)
    {
        var eastern = TimeZoneInfo.ConvertTime(from, EasternTime);
        var date = DateOnly.FromDateTime(eastern.DateTime);
        var time = TimeOnly.FromDateTime(eastern.DateTime);
        
        // If before regular open today and it's a trading day
        if (IsTradingDay(date) && time < RegularOpen)
        {
            return ToEasternDateTimeOffset(date, RegularOpen);
        }
        
        // Otherwise, find next trading day
        var nextDay = GetNextTradingDay(date);
        return ToEasternDateTimeOffset(nextDay, RegularOpen);
    }
    
    public DateTimeOffset GetNextMarketClose(DateTimeOffset from)
    {
        var eastern = TimeZoneInfo.ConvertTime(from, EasternTime);
        var date = DateOnly.FromDateTime(eastern.DateTime);
        var time = TimeOnly.FromDateTime(eastern.DateTime);
        
        if (IsTradingDay(date))
        {
            var closeTime = GetEarlyCloseTime(date) ?? RegularClose;
            if (time < closeTime)
            {
                return ToEasternDateTimeOffset(date, closeTime);
            }
        }
        
        var nextDay = GetNextTradingDay(date);
        var nextCloseTime = GetEarlyCloseTime(nextDay) ?? RegularClose;
        return ToEasternDateTimeOffset(nextDay, nextCloseTime);
    }
    
    public TimeSpan GetTimeUntilOpen(DateTimeOffset from)
    {
        if (GetStatus(from) == MarketStatus.Open)
            return TimeSpan.Zero;
        
        var nextOpen = GetNextMarketOpen(from);
        return nextOpen - from;
    }
    
    private static DateTimeOffset ToEasternDateTimeOffset(DateOnly date, TimeOnly time)
    {
        var dt = date.ToDateTime(time);
        return new DateTimeOffset(dt, EasternTime.GetUtcOffset(dt));
    }
    
    #endregion
}
```

---

## Quartz.NET Integration

### MarketHolidayCalendar.cs

```csharp
using Quartz;
using Quartz.Impl.Calendar;

namespace Atlas.Calendar.Quartz;

/// <summary>
/// Quartz calendar that excludes market holidays.
/// Attach to triggers to skip job execution on market closures.
/// </summary>
public sealed class MarketHolidayCalendar : BaseCalendar
{
    private readonly IMarketCalendar _marketCalendar;
    
    public const string CalendarName = "market-holidays";
    
    public MarketHolidayCalendar(IMarketCalendar marketCalendar)
        : base(null)
    {
        _marketCalendar = marketCalendar;
    }
    
    public MarketHolidayCalendar(IMarketCalendar marketCalendar, ICalendar? baseCalendar)
        : base(baseCalendar)
    {
        _marketCalendar = marketCalendar;
    }
    
    /// <summary>
    /// Returns true if the given time is included (should fire), 
    /// false if excluded (should skip).
    /// </summary>
    public override bool IsTimeIncluded(DateTimeOffset timeUtc)
    {
        // Check base calendar first
        if (!base.IsTimeIncluded(timeUtc))
            return false;
        
        var date = DateOnly.FromDateTime(timeUtc.LocalDateTime);
        
        // Exclude market holidays
        return !_marketCalendar.IsMarketClosed(date);
    }
    
    public override DateTimeOffset GetNextIncludedTimeUtc(DateTimeOffset timeUtc)
    {
        var nextTime = timeUtc.AddMilliseconds(1);
        
        while (!IsTimeIncluded(nextTime))
        {
            // Move to start of next day
            nextTime = nextTime.Date.AddDays(1).ToUniversalTime();
            
            // Safety limit
            if (nextTime > timeUtc.AddDays(14))
                throw new InvalidOperationException("Could not find next included time within 14 days");
        }
        
        return nextTime;
    }
    
    public override object Clone()
    {
        return new MarketHolidayCalendar(_marketCalendar, CalendarBase);
    }
}
```

### TradingHoursCalendar.cs

```csharp
using Quartz;
using Quartz.Impl.Calendar;

namespace Atlas.Calendar.Quartz;

/// <summary>
/// Quartz calendar that only allows execution during market hours.
/// Use for real-time data collection that should pause overnight.
/// </summary>
public sealed class TradingHoursCalendar : BaseCalendar
{
    private readonly IMarketCalendar _marketCalendar;
    private readonly bool _includePreMarket;
    private readonly bool _includeAfterHours;
    
    public const string CalendarName = "trading-hours";
    
    public TradingHoursCalendar(
        IMarketCalendar marketCalendar,
        bool includePreMarket = false,
        bool includeAfterHours = false)
        : base(null)
    {
        _marketCalendar = marketCalendar;
        _includePreMarket = includePreMarket;
        _includeAfterHours = includeAfterHours;
    }
    
    public override bool IsTimeIncluded(DateTimeOffset timeUtc)
    {
        if (!base.IsTimeIncluded(timeUtc))
            return false;
        
        var status = _marketCalendar.GetStatus(timeUtc);
        
        return status switch
        {
            MarketStatus.Open => true,
            MarketStatus.PreMarket => _includePreMarket,
            MarketStatus.AfterHours => _includeAfterHours,
            _ => false
        };
    }
    
    public override DateTimeOffset GetNextIncludedTimeUtc(DateTimeOffset timeUtc)
    {
        var nextTime = timeUtc.AddMinutes(1);
        var attempts = 0;
        
        while (!IsTimeIncluded(nextTime) && attempts < 10080) // 1 week in minutes
        {
            nextTime = nextTime.AddMinutes(1);
            attempts++;
        }
        
        return nextTime;
    }
    
    public override object Clone()
    {
        return new TradingHoursCalendar(_marketCalendar, _includePreMarket, _includeAfterHours);
    }
}
```

---

## DI Registration

### ServiceCollectionExtensions.cs

```csharp
using Microsoft.Extensions.DependencyInjection;
using Microsoft.Extensions.DependencyInjection.Extensions;

namespace Atlas.Calendar.Extensions;

public static class ServiceCollectionExtensions
{
    /// <summary>
    /// Add Atlas.Calendar services to the DI container.
    /// </summary>
    public static IServiceCollection AddAtlasCalendar(this IServiceCollection services)
    {
        // Register primary calendar implementation
        services.TryAddSingleton<NyseMarketCalendar>();
        services.TryAddSingleton<IMarketCalendar>(sp => sp.GetRequiredService<NyseMarketCalendar>());
        services.TryAddSingleton<ITradingHoursProvider>(sp => sp.GetRequiredService<NyseMarketCalendar>());
        
        return services;
    }
    
    /// <summary>
    /// Add Quartz calendar integrations.
    /// Call this after AddQuartz().
    /// </summary>
    public static IServiceCollection AddAtlasQuartzCalendars(this IServiceCollection services)
    {
        services.AddSingleton<MarketHolidayCalendar>();
        services.AddSingleton<TradingHoursCalendar>();
        
        return services;
    }
}
```

### Quartz Configuration Example

```csharp
// In Program.cs or Startup
services.AddQuartz(q =>
{
    // ... job configurations ...
});

services.AddAtlasCalendar();
services.AddAtlasQuartzCalendars();

// In a hosted service or startup task
public class QuartzCalendarSetup : IHostedService
{
    private readonly ISchedulerFactory _schedulerFactory;
    private readonly MarketHolidayCalendar _holidayCalendar;
    private readonly TradingHoursCalendar _tradingHoursCalendar;
    
    public async Task StartAsync(CancellationToken cancellationToken)
    {
        var scheduler = await _schedulerFactory.GetScheduler(cancellationToken);
        
        // Register calendars
        await scheduler.AddCalendar(
            MarketHolidayCalendar.CalendarName,
            _holidayCalendar,
            replace: true,
            updateTriggers: true,
            cancellationToken);
            
        await scheduler.AddCalendar(
            TradingHoursCalendar.CalendarName,
            _tradingHoursCalendar,
            replace: true,
            updateTriggers: true,
            cancellationToken);
    }
    
    public Task StopAsync(CancellationToken cancellationToken) => Task.CompletedTask;
}
```

### Job Trigger Example

```csharp
// FinnhubCollector quote polling - skip holidays
var quoteTrigger = TriggerBuilder.Create()
    .WithIdentity("quote-trigger", "finnhub")
    .WithCronSchedule("0 */5 * ? * MON-FRI") // Every 5 min weekdays
    .ModifiedByCalendar(MarketHolidayCalendar.CalendarName) // Skip holidays
    .Build();

// Real-time VIX polling - only during market hours
var vixTrigger = TriggerBuilder.Create()
    .WithIdentity("vix-trigger", "finnhub")
    .WithCronSchedule("0 */1 * ? * MON-FRI") // Every minute weekdays
    .ModifiedByCalendar(TradingHoursCalendar.CalendarName) // Only market hours
    .Build();
```

---

## Usage Examples

### Basic Holiday Check

```csharp
public class SomeService
{
    private readonly IMarketCalendar _calendar;
    
    public void ProcessData(DateOnly date)
    {
        if (_calendar.IsMarketClosed(date))
        {
            _logger.LogInformation("Skipping {Date} - market closed", date);
            return;
        }
        
        // Process...
    }
}
```

### Trading Day Navigation

```csharp
// Get previous trading day for T+1 settlement
var settlementDate = _calendar.GetNextTradingDay(tradeDate);

// Get last 20 trading days for moving average
var twentyDaysAgo = _calendar.GetTradingDayOffset(today, -20);

// Count business days for reporting
var tradingDays = _calendar.CountTradingDays(monthStart, monthEnd);
```

### Adaptive Polling Interval

```csharp
public class QuotePollingWorker : BackgroundService
{
    private readonly IMarketCalendar _calendar;
    private readonly ITradingHoursProvider _tradingHours;
    
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            var status = _calendar.GetCurrentStatus();
            
            var interval = status switch
            {
                MarketStatus.Open => TimeSpan.FromSeconds(15),      // Aggressive during hours
                MarketStatus.PreMarket => TimeSpan.FromMinutes(1),  // Less frequent pre-market
                MarketStatus.AfterHours => TimeSpan.FromMinutes(5), // Minimal after hours
                _ => _tradingHours.GetTimeUntilOpen(DateTimeOffset.UtcNow) // Sleep until open
            };
            
            await CollectQuotesAsync(stoppingToken);
            await Task.Delay(interval, stoppingToken);
        }
    }
}
```

---

## Phase 2: Economic Calendar (Future)

### FinnhubEconomicCalendar.cs

```csharp
namespace Atlas.Calendar.Providers;

/// <summary>
/// IEconomicCalendar backed by Finnhub /calendar/economic endpoint.
/// Phase 2 implementation - requires FinnhubCollector dependency.
/// </summary>
public sealed class FinnhubEconomicCalendar : IEconomicCalendar
{
    private readonly IFinnhubApiClient _finnhubClient;
    private readonly IMemoryCache _cache;
    private readonly ILogger<FinnhubEconomicCalendar> _logger;
    
    private static readonly TimeSpan CacheDuration = TimeSpan.FromHours(6);
    
    public async Task<IReadOnlyList<EconomicEvent>> GetUpcomingEventsAsync(
        int daysAhead = 30,
        CancellationToken cancellationToken = default)
    {
        var cacheKey = $"economic-events-{daysAhead}";
        
        if (_cache.TryGetValue<IReadOnlyList<EconomicEvent>>(cacheKey, out var cached))
            return cached!;
        
        var from = DateOnly.FromDateTime(DateTime.UtcNow);
        var to = from.AddDays(daysAhead);
        
        var finnhubEvents = await _finnhubClient.GetEconomicCalendarAsync(from, to, cancellationToken);
        
        var events = finnhubEvents
            .Where(e => e.Country == "US")
            .Select(MapToEconomicEvent)
            .OrderBy(e => e.EventTime)
            .ToList();
        
        _cache.Set(cacheKey, events, CacheDuration);
        return events;
    }
    
    // ... other methods ...
    
    private static EconomicEvent MapToEconomicEvent(FinnhubEconomicCalendarEvent source)
    {
        return new EconomicEvent
        {
            EventId = $"finnhub-{source.EventTime:yyyyMMdd}-{source.Event.GetHashCode():X8}",
            Name = source.Event,
            Type = InferEventType(source.Event),
            EventTime = source.EventTime,
            Country = source.Country,
            Impact = ParseImpact(source.Impact),
            Estimate = source.Estimate,
            Actual = source.Actual,
            Previous = source.Previous,
            Unit = source.Unit,
            Source = "Finnhub",
            CollectedAt = DateTimeOffset.UtcNow
        };
    }
    
    private static EconomicEventType InferEventType(string eventName)
    {
        var lower = eventName.ToLowerInvariant();
        
        if (lower.Contains("fomc") || lower.Contains("fed funds"))
            return EconomicEventType.FedMeeting;
        if (lower.Contains("fed") && lower.Contains("minutes"))
            return EconomicEventType.FedMinutes;
        if (lower.Contains("cpi"))
            return EconomicEventType.CpiRelease;
        if (lower.Contains("ppi"))
            return EconomicEventType.PpiRelease;
        if (lower.Contains("nonfarm") || lower.Contains("unemployment"))
            return EconomicEventType.EmploymentReport;
        if (lower.Contains("gdp"))
            return EconomicEventType.GdpRelease;
        if (lower.Contains("retail sales"))
            return EconomicEventType.RetailSales;
        if (lower.Contains("ism") && lower.Contains("manufacturing"))
            return EconomicEventType.IsmManufacturing;
        if (lower.Contains("ism") && lower.Contains("services"))
            return EconomicEventType.IsmServices;
        if (lower.Contains("consumer sentiment") || lower.Contains("umich"))
            return EconomicEventType.ConsumerSentiment;
        
        return EconomicEventType.Other;
    }
    
    private static EconomicImpact ParseImpact(string impact) => impact.ToLowerInvariant() switch
    {
        "high" => EconomicImpact.High,
        "medium" => EconomicImpact.Medium,
        _ => EconomicImpact.Low
    };
}
```

---

## Tasks

| # | Task | Est | Status |
|---|------|-----|--------|
| 1 | Create Atlas.Calendar project structure | 30m | ⬜ |
| 2 | Implement Models (MarketHoliday, TradingSession, enums) | 30m | ⬜ |
| 3 | Implement StaticNyseHolidays (Easter calc, early closes) | 45m | ⬜ |
| 4 | Implement NyseMarketCalendar (IMarketCalendar) | 1h | ⬜ |
| 5 | Implement ITradingHoursProvider methods | 30m | ⬜ |
| 6 | Implement MarketHolidayCalendar (Quartz) | 30m | ⬜ |
| 7 | Implement TradingHoursCalendar (Quartz) | 30m | ⬜ |
| 8 | Add ServiceCollectionExtensions | 15m | ⬜ |
| 9 | Unit tests: Holiday calculations (2024-2026) | 1h | ⬜ |
| 10 | Unit tests: Trading day navigation | 45m | ⬜ |
| 11 | Unit tests: Quartz calendar behavior | 45m | ⬜ |
| 12 | Integration: Wire into FinnhubCollector | 30m | ⬜ |
| 13 | Integration: Wire into FredCollector | 30m | ⬜ |
| 14 | Documentation: Update ARCHITECTURE.md | 15m | ⬜ |

**Total Estimate**: ~8-10 hours

---

## Success Criteria

- [ ] All NYSE holidays 2024-2026 correctly identified
- [ ] Early close days properly handled (1 PM ET close)
- [ ] Good Friday calculated correctly (non-federal holiday)
- [ ] Quartz triggers skip market holidays
- [ ] FinnhubCollector uses adaptive polling based on market status
- [ ] No external API calls for holiday data (offline-first)
- [ ] Unit test coverage >90% for calendar logic

---

## Future Enhancements

- **Phase 2**: `IEconomicCalendar` backed by Finnhub
- **Phase 3**: Multi-exchange support (LSE, TSE, etc.)
- **Phase 4**: Admin API for manual holiday overrides
- **Phase 5**: Promote to standalone CalendarService microservice if needed
