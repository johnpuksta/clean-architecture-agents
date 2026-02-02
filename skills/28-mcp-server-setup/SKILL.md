# MCP Server Setup

Implements Model Context Protocol (MCP) server infrastructure for .NET Clean Architecture applications using Server-Sent Events (SSE).

## Package Requirements

```xml
<PackageReference Include="ModelContextProtocol.AspNetCore" Version="0.1.0-preview.10" />
```

## Project Structure

```
src/
  YourProject.Mcp/
    Program.cs
    Extensions/ServiceCollectionExtensions.cs
    Tools/YourTools.cs
    appsettings.json
```

## Implementation

### 1. Create Project

**YourProject.Mcp.csproj**

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="ModelContextProtocol.AspNetCore" Version="0.1.0-preview.10" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="..\YourProject.Application\YourProject.Application.csproj" />
    <ProjectReference Include="..\YourProject.Infrastructure\YourProject.Infrastructure.csproj" />
  </ItemGroup>
</Project>
```

### 2. Configure appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "ConnectionStrings": {
    "DefaultConnection": "Host=localhost;Database=yourdb;Username=postgres;Password=postgres"
  },
  "Mcp": {
    "ServerName": "yourproject",
    "ServerVersion": "1.0.0",
    "SseEndpoint": "/mcp",
    "MaxConcurrentConnections": 100
  },
  "Kestrel": {
    "Endpoints": {
      "Http": {
        "Url": "http://localhost:5001"
      }
    }
  }
}
```

### 3. Extension Method (C# 13)

**Extensions/ServiceCollectionExtensions.cs**

```csharp
namespace YourProject.Mcp.Extensions;

public static class ServiceCollectionExtensions
{
    extension(IServiceCollection services)
    {
        public IServiceCollection AddMcpServerWithTools()
        {
            services.AddMcpServer()
                .WithToolsFromAssembly(typeof(ServiceCollectionExtensions).Assembly);

            return services;
        }
    }
}
```

### 4. Program.cs

```csharp
using YourProject.Application.Extensions;
using YourProject.Infrastructure.Extensions;
using YourProject.Mcp.Extensions;

var builder = WebApplication.CreateBuilder(args);

builder.Logging.ClearProviders();
builder.Logging.AddConfiguration(builder.Configuration.GetSection("Logging"));
builder.Logging.AddConsole();

if (builder.Environment.IsDevelopment())
{
    builder.Logging.AddDebug();
}

builder.Services.AddApplication();
builder.Services.AddInfrastructure(builder.Configuration);
builder.Services.AddMcpServerWithTools();

var app = builder.Build();

var logger = app.Services.GetRequiredService<ILogger<Program>>();
logger.LogInformation("Starting MCP Server...");

app.MapMcp();

app.MapGet("/health", () => Results.Ok(new { status = "healthy", timestamp = DateTime.UtcNow }))
    .WithName("HealthCheck");

app.Run();
```

### 5. Implement Tools

**Tools/ExampleTools.cs**

```csharp
using System.ComponentModel;
using System.Text.Json;
using YourProject.Application.Commands;
using YourProject.Application.Queries;
using Kommand.Abstractions;
using ModelContextProtocol.Server;

namespace YourProject.Mcp.Tools;

[McpServerToolType]
public class ExampleTools
{
    private readonly IMediator _mediator;

    public ExampleTools(IMediator mediator)
    {
        _mediator = mediator;
    }

    [McpServerTool]
    [Description("Get data by ID")]
    public async Task<string> GetData(
        [Description("The ID to retrieve")] string id,
        CancellationToken cancellationToken = default)
    {
        var parsedId = Guid.Parse(id);
        var query = new GetDataQuery(parsedId);
        var result = await _mediator.QueryAsync(query, cancellationToken);

        if (result.IsFailure)
        {
            return $"Error: {result.Error.Description}";
        }

        return JsonSerializer.Serialize(result.Value, new JsonSerializerOptions { WriteIndented = true });
    }

    [McpServerTool]
    [Description("Create item with optional metadata")]
    public async Task<string> CreateItem(
        [Description("Name of the item")] string name,
        [Description("Description")] string description,
        [Description("Optional metadata as JSON object")] string? metadataJson = null,
        CancellationToken cancellationToken = default)
    {
        Dictionary<string, string>? metadata = null;
        if (!string.IsNullOrEmpty(metadataJson))
        {
            metadata = JsonSerializer.Deserialize<Dictionary<string, string>>(metadataJson);
        }

        var command = new CreateItemCommand(name, description, metadata);
        var result = await _mediator.SendAsync(command, cancellationToken);

        if (result.IsFailure)
        {
            return $"Error: {result.Error.Description}";
        }

        return JsonSerializer.Serialize(new
        {
            id = result.Value,
            message = "Item created successfully"
        }, new JsonSerializerOptions { WriteIndented = true });
    }
}
```

## Tool Patterns

### Basic Structure

```csharp
[McpServerToolType]
public class YourTools
{
    private readonly IMediator _mediator;

    public YourTools(IMediator mediator)
    {
        _mediator = mediator;
    }

    [McpServerTool]
    [Description("Clear description")]
    public async Task<string> ToolName(
        [Description("Parameter description")] string requiredParam,
        [Description("Optional param")] string? optionalParam = null,
        CancellationToken cancellationToken = default)
    {
        // 1. Parse inputs
        // 2. Create command/query
        // 3. Send via mediator
        // 4. Handle result
        // 5. Return JSON
    }
}
```

### Required Attributes

- `[McpServerToolType]` - On classes
- `[McpServerTool]` - On methods (must return `Task<string>`)
- `[Description("...")]` - On methods and parameters (from `System.ComponentModel`)

### Parameter Patterns

```csharp
// Simple parameters
[Description("The ID")] string id

// Optional with default
[Description("Filter")] string? filter = null

// Complex types via JSON
[Description("Tags as JSON array")] string? tagsJson = null

// Parse JSON parameters
List<string>? tags = null;
if (!string.IsNullOrEmpty(tagsJson))
{
    tags = JsonSerializer.Deserialize<List<string>>(tagsJson);
}
```

### Return Patterns

```csharp
// Success - return data
return JsonSerializer.Serialize(result.Value, new JsonSerializerOptions { WriteIndented = true });

// Failure - return error
if (result.IsFailure)
{
    return $"Error: {result.Error.Description}";
}

// Success with metadata
return JsonSerializer.Serialize(new
{
    id = result.Value,
    message = "Success"
}, new JsonSerializerOptions { WriteIndented = true });
```

## Complex Example

```csharp
using System.ComponentModel;
using System.Text.Json;
using YourProject.Application.Commands.Tasks;
using YourProject.Application.Queries;
using YourProject.Domain.Enums;
using Kommand.Abstractions;
using ModelContextProtocol.Server;

namespace YourProject.Mcp.Tools;

[McpServerToolType]
public class TaskTools
{
    private readonly IMediator _mediator;

    public TaskTools(IMediator mediator)
    {
        _mediator = mediator;
    }

    [McpServerTool]
    [Description("Create task with priority and tags")]
    public async Task<string> CreateTask(
        [Description("Task title")] string title,
        [Description("Detailed description")] string description,
        [Description("Priority: Low, Medium, High")] string priority,
        [Description("Tags as JSON array e.g. [\"bug\",\"urgent\"]")] string? tagsJson = null,
        CancellationToken cancellationToken = default)
    {
        if (!Enum.TryParse<TaskPriority>(priority, ignoreCase: true, out var taskPriority))
        {
            return "Error: Invalid priority. Must be Low, Medium, or High";
        }

        List<string>? tags = null;
        if (!string.IsNullOrEmpty(tagsJson))
        {
            try
            {
                tags = JsonSerializer.Deserialize<List<string>>(tagsJson);
            }
            catch (JsonException ex)
            {
                return $"Error: Invalid tags JSON - {ex.Message}";
            }
        }

        var command = new CreateTaskCommand(title, description, taskPriority, tags);
        var result = await _mediator.SendAsync(command, cancellationToken);

        if (result.IsFailure)
        {
            return $"Error: {result.Error.Description}";
        }

        return JsonSerializer.Serialize(new
        {
            taskId = result.Value,
            title,
            priority = taskPriority.ToString(),
            tags = tags ?? new List<string>(),
            createdAt = DateTime.UtcNow
        }, new JsonSerializerOptions { WriteIndented = true });
    }

    [McpServerTool]
    [Description("Get tasks with optional filters")]
    public async Task<string> GetTasks(
        [Description("Status: Pending, InProgress, Completed")] string? status = null,
        [Description("Priority: Low, Medium, High")] string? priority = null,
        CancellationToken cancellationToken = default)
    {
        TaskStatus? taskStatus = null;
        if (!string.IsNullOrEmpty(status) && !Enum.TryParse<TaskStatus>(status, ignoreCase: true, out var parsed))
        {
            return "Error: Invalid status";
        }
        else if (!string.IsNullOrEmpty(status))
        {
            taskStatus = parsed;
        }

        TaskPriority? taskPriority = null;
        if (!string.IsNullOrEmpty(priority) && !Enum.TryParse<TaskPriority>(priority, ignoreCase: true, out var parsedPrio))
        {
            return "Error: Invalid priority";
        }
        else if (!string.IsNullOrEmpty(priority))
        {
            taskPriority = parsedPrio;
        }

        var query = new GetTasksQuery(taskStatus, taskPriority);
        var result = await _mediator.QueryAsync(query, cancellationToken);

        if (result.IsFailure)
        {
            return $"Error: {result.Error.Description}";
        }

        return JsonSerializer.Serialize(result.Value, new JsonSerializerOptions { WriteIndented = true });
    }
}
```

## Key Points

1. **Tools auto-discovered** via `WithToolsFromAssembly()`
2. **Always use IMediator** - Never call repositories/domain directly
3. **Return JSON strings** - All tools return `Task<string>`
4. **Handle Result pattern** - Check `IsFailure` and return errors as strings
5. **Validate inputs** - Parse and validate before processing
6. **Complex params as JSON** - Deserialize JSON string parameters
7. **Description required** - Every tool and parameter needs `[Description]`
8. **C# 13 extensions** - Use `extension` keyword for DI

## Running

```bash
dotnet run --project src/YourProject.Mcp
```

Connect AI agents to: `http://localhost:5001/mcp`

## Related Skills

- 02-cqrs-command-generator
- 03-cqrs-query-generator
- 24-logging-configuration
- 17-health-checks
