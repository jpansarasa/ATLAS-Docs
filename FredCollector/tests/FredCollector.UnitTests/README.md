# Unit Tests

This directory contains unit tests for the FredCollector project.

## Test Structure

- **Domain/**: Tests for domain models (SeriesConfig, ThresholdAlert, FredObservation)
- **BusinessLogic/**: Tests for business logic (threshold evaluation, collection scheduling)
- **Services/**: Tests for services (rate limiter - currently skipped until implementation exists)
- **Grpc/**: Tests for gRPC event streaming components
  - EventRepositoryTests.cs: Query methods, filtering, chronological ordering
  - EventStreamServiceTests.cs: gRPC endpoints, limit handling, filter passing
- **Publishers/**: Tests for event publishing
  - EventPublisherTests.cs: Database persistence, serialization, batch operations

## Running Tests

```bash
# Run all tests
dotnet test

# Run specific test project
dotnet test tests/FredCollector.UnitTests

# Run with verbose output
dotnet test --verbosity normal

# Run specific test class
dotnet test --filter "FullyQualifiedName~SeriesConfigTests"
```

## Test Coverage

### Domain Models
- ✅ SeriesConfig: Initialization, active state, threshold configuration
- ✅ ThresholdAlert: State management, threshold evaluation
- ✅ FredObservation: Validation, revision tracking, calculations

### Business Logic
- ✅ ThresholdEvaluation: All threshold directions, edge cases
- ✅ CollectionScheduling: Frequency-based scheduling, due date calculation

### Services
- ⏸️ RateLimiter: Tests created but skipped (requires implementation)

### gRPC Event Streaming (E11)
- ✅ EventPublisher: Database storage, JSON serialization, batch insert, error types
- ✅ EventRepository: Chronological ordering, event type filtering, series filtering, aggregations
- ✅ EventStreamService: GetEventsSince, GetEventsBetween, GetHealth, limit handling, filter passing

## Notes

- Extension methods are defined in the same namespace as the tests
- Business logic helper classes (ThresholdEvaluator, CollectionScheduler) are defined in test files
- Rate limiter tests are skipped until TokenBucketRateLimiter implementation exists

