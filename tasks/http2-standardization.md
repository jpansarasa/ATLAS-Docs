# HTTP/2 Standardization

## Overview

Standardize all internal service communication on HTTP/2 minimum. Currently, several services fall back to HTTP/1.1 unnecessarily.

## Current State

### Servers (Kestrel)

| Service | Port | Current Protocol | Issue |
|---------|------|------------------|-------|
| SecMaster | 8080 | Http1AndHttp2 | Should be Http2 only |
| OfrCollector | 8080 | Default (Http1) | Should be Http2 |
| OfrCollector | 5001 | Http2 | Correct |
| FredCollector | 8080 | Default (Http1) | Should be Http2 |
| FredCollector | 5001 | Http2 | Correct |
| FinnhubCollector | 5001 | Http2 | Correct |
| AlphaVantageCollector | - | appsettings Http2 | Verify |
| NasdaqCollector | - | appsettings Http2 | Verify |

### Clients (HttpClient)

| Location | Current | Issue |
|----------|---------|-------|
| SecMaster collector clients | Default (Http1) | Missing DefaultRequestVersion |
| SecMaster OllamaClient | Default (Http1) | Missing DefaultRequestVersion |
| SecMasterMcp SecMasterClient | Default (Http1) | Missing DefaultRequestVersion |
| ThresholdEngine clients | Http2 | Correct (gRPC) |
| Events clients | Http2 | Correct (gRPC) |

## Tasks

### Phase 1: HttpClient Configuration

- [ ] SecMaster/src/DependencyInjection.cs - Add HTTP/2 to all HttpClient registrations
- [ ] SecMasterMcp/src/Program.cs - Add HTTP/2 to SecMasterClient
- [ ] Verify other MCP servers (FredCollectorMcp, etc.)

```csharp
services.AddHttpClient<IFredCollectorClient, FredCollectorClient>((sp, client) =>
{
    client.BaseAddress = new Uri(collectorOptions.FredCollectorUrl);
    client.Timeout = TimeSpan.FromSeconds(30);
    client.DefaultRequestVersion = HttpVersion.Version20;
    client.DefaultVersionPolicy = HttpVersionPolicy.RequestVersionOrHigher;
})
```

### Phase 2: Kestrel Server Configuration

- [ ] SecMaster/src/Program.cs - Change `Http1AndHttp2` to `Http2`
- [ ] OfrCollector/src/Program.cs - Set port 8080 to Http2
- [ ] FredCollector/src/Program.cs - Set port 8080 to Http2
- [ ] Remove misleading "HTTP/1.1 for health checks" comments
- [ ] Verify AlphaVantageCollector appsettings.json
- [ ] Verify NasdaqCollector appsettings.json

```csharp
builder.WebHost.UseKestrel(options =>
{
    options.ListenAnyIP(8080, listenOptions =>
    {
        listenOptions.Protocols = HttpProtocols.Http2;
    });
});
```

### Phase 3: Verification

- [ ] Verify health checks work with HTTP/2 (Kubernetes probes)
- [ ] Verify Grafana health check dashboard
- [ ] Test smoke tests pass
- [ ] Verify gRPC services still work

### Phase 4: Documentation

- [ ] Update SecMaster/README.md port documentation
- [ ] Update any architecture docs mentioning HTTP/1.1
- [ ] Remove STATE.md references to HTTP/1.1

## HTTP/3 Future Consideration

HTTP/3 is not prioritized for internal services because:

1. **Requires TLS** - Currently using cleartext h2c internally
2. **UDP-based (QUIC)** - Container networks are reliable, benefits minimal
3. **Connection migration** - Container IPs are stable
4. **.NET 9 support** - Still experimental

HTTP/3 should be considered for:
- External-facing MCP servers (ports 31xx)
- Future public API endpoints
- Mobile/unreliable network clients

## References

- [Kestrel HTTP/2 Configuration](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/kestrel/http2)
- [HttpClient HTTP/2](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient-guidelines#http2)
- [.NET HTTP/3 Support](https://learn.microsoft.com/en-us/dotnet/core/extensions/httpclient-http3)
