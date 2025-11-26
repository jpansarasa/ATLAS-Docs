# Task: Enable OpenAPI/Swagger JSON Export

## Problem
Manually maintained API documentation drifts from implementation. We have Swagger configured but:
1. Only enabled in Development mode
2. No static export for offline reference

## Solution

### 1. Enable swagger.json in all environments

**FredCollector.Api/Program.cs** - Change:
```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    logger.LogWarning("Swagger UI enabled at /swagger");
}
```

To:
```csharp
// Always serve OpenAPI spec (JSON only)
app.UseSwagger();

// Only serve interactive UI in development
if (app.Environment.IsDevelopment())
{
    app.UseSwaggerUI();
    logger.LogWarning("Swagger UI enabled at /swagger");
}
```

**ThresholdEngine.Service/Program.cs** - Same change (if Swagger configured)

### 2. Add build-time OpenAPI export

**FredCollector.Api/FredCollector.Api.csproj** - Add:
```xml
<PropertyGroup>
  <GenerateDocumentationFile>true</GenerateDocumentationFile>
</PropertyGroup>

<Target Name="GenerateOpenApiSpec" AfterTargets="Build" Condition="'$(GenerateOpenApi)' == 'true'">
  <Exec Command="dotnet swagger tofile --output $(SolutionDir)docs/openapi/fredcollector-v1.json $(OutputPath)$(AssemblyName).dll v1" 
        WorkingDirectory="$(ProjectDir)" />
</Target>
```

**ThresholdEngine.Service.csproj** - Same pattern:
```xml
<Target Name="GenerateOpenApiSpec" AfterTargets="Build" Condition="'$(GenerateOpenApi)' == 'true'">
  <Exec Command="dotnet swagger tofile --output $(SolutionDir)docs/openapi/thresholdengine-v1.json $(OutputPath)$(AssemblyName).dll v1" 
        WorkingDirectory="$(ProjectDir)" />
</Target>
```

### 3. Add Swashbuckle CLI tool

```bash
dotnet tool install --global Swashbuckle.AspNetCore.Cli
```

Or add to tool manifest:
```bash
dotnet new tool-manifest
dotnet tool install Swashbuckle.AspNetCore.Cli
```

### 4. Generate on demand

```bash
# Generate OpenAPI specs
dotnet build -p:GenerateOpenApi=true
```

### 5. Output location

```
ATLAS/
├── docs/
│   └── openapi/
│       ├── fredcollector-v1.json    # Auto-generated
│       └── thresholdengine-v1.json  # Auto-generated
```

## Benefits

1. **Runtime access**: MCP servers can fetch `/swagger/v1/swagger.json` directly
2. **Build-time export**: Static JSON for offline reference, committed to repo
3. **Never drifts**: Generated from actual code
4. **Claude can read**: Both live endpoint and static file are machine-readable

## MCP Integration

MCP servers can:
```csharp
// Option 1: Fetch live spec
var spec = await httpClient.GetStringAsync("http://mercury:5001/swagger/v1/swagger.json");

// Option 2: Read embedded resource (if bundled)
var spec = Assembly.GetExecutingAssembly().GetManifestResourceStream("fredcollector-v1.json");
```

## Verification

After implementation:
```bash
# Test runtime endpoint
curl http://mercury:5001/swagger/v1/swagger.json | jq .info

# Test build-time generation
dotnet build -p:GenerateOpenApi=true
cat docs/openapi/fredcollector-v1.json | jq .paths
```

## Files to Modify

1. `FredCollector/src/FredCollector.Api/Program.cs` - Enable swagger in all envs
2. `FredCollector/src/FredCollector.Api/FredCollector.Api.csproj` - Add build target
3. `ThresholdEngine/src/ThresholdEngine.Service/Program.cs` - Enable swagger (if not already)
4. `ThresholdEngine/src/ThresholdEngine.Service/ThresholdEngine.Service.csproj` - Add build target
5. Create `docs/openapi/.gitkeep`

## Delete After Implementation

These manually maintained files can be deleted:
- `docs/FREDCOLLECTOR_API.md`
- `docs/THRESHOLDENGINE_API.md`

The OpenAPI JSON is the source of truth.
