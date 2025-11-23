# ThresholdEngine Unit Tests

## Running Tests

### In Dev Container

```bash
# Inside VS Code Dev Container (recommended)
cd /workspace
dotnet test

# Or use the test script
./scripts/run-tests.sh
```

### From Host (if .NET SDK available)

```bash
cd ThresholdEngine
dotnet test
```

## Test Structure

### Configuration Tests

- **PatternConfigurationLoaderTests**: Tests for loading and validating pattern configurations from JSON files
- **PatternConfigurationRepositoryTests**: Tests for in-memory pattern storage and retrieval
- **PatternConfigurationWatcherTests**: Tests for hot reload functionality (basic lifecycle tests)
- **ConfigurationAuditRepositoryTests**: Tests for audit logging
- **PatternConfigurationLoaderIntegrationTests**: Integration-style tests with real JSON files

## Test Coverage

### PatternConfigurationLoader (15+ tests)
- ✅ Empty directory handling
- ✅ Valid pattern loading
- ✅ Disabled pattern exclusion
- ✅ Invalid JSON handling
- ✅ Duplicate PatternId detection
- ✅ Recursive directory scanning
- ✅ Validation (null, missing fields, invalid formats)
- ✅ Category filtering

### PatternConfigurationRepository (12+ tests)
- ✅ GetByIdAsync (existing, non-existent, null)
- ✅ GetByCategoryAsync (filtering, disabled exclusion)
- ✅ GetByRegimeAsync (regime filtering)
- ✅ GetAllEnabledAsync (enabled only)
- ✅ ReplaceAllAsync (duplicates, null, atomic replacement)
- ✅ GetMetricsAsync (counts and categories)
- ✅ Error tracking methods

### PatternConfigurationWatcher (4 tests)
- ✅ StartAsync/StopAsync lifecycle
- ✅ ConfigurationReloaded event
- ✅ Dispose cleanup
- ⚠️ File change detection requires integration tests

### ConfigurationAuditRepository (5 tests)
- ✅ SaveAsync (valid, null)
- ✅ GetRecentAsync (ordering, count limits)
- ✅ GetSinceAsync (timestamp filtering)
- ✅ Max log eviction (1000 limit)

## Expected Test Results

When running `dotnet test`, you should see:
- **Total Tests**: ~36+ tests
- **Passed**: All tests should pass
- **Failed**: 0
- **Skipped**: 0

## Troubleshooting

### Common Issues

1. **Missing dependencies**: Run `dotnet restore` first
2. **Build errors**: Run `dotnet build` to see compilation errors
3. **Test discovery failures**: Ensure test project references are correct
4. **File system permissions**: Tests create temp directories, ensure write permissions

## Next Steps

After E2 completion, add integration tests for:
- FileSystemWatcher file change detection
- End-to-end configuration load → validate → reload cycle
- Error tracking metrics verification

