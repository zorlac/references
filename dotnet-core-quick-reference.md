# .NET Core Quick Reference for Java Developers

## Table of Contents

- [IDE Options](#ide-options)
- [Solution vs Project](#solution-vs-project)
- [Project Structure](#project-structure)
- [Controllers](#controllers)
- [Routes](#routes)
- [Return Types](#return-types)
- [Async/Await](#asyncawait)
- [Configuration Files](#configuration-files)
- [Dependencies (NuGet)](#dependencies-nuget)
- [Swagger UI](#swagger-ui)
- [Environment-Based Config](#environment-based-config)

---

## IDE Options

| IDE | Notes |
|-----|-------|
| **Visual Studio** | Full IDE, Community edition free |
| **JetBrains Rider** | IntelliJ-like experience, paid |
| **VS Code + C# Dev Kit** | Lightweight, free |

---

## Solution vs Project

| Concept | .NET | Java Equivalent |
|---------|------|-----------------|
| **Solution** (`.sln`) | Container for multiple projects | Maven parent pom |
| **Project** (`.csproj`) | Single buildable unit | Maven module |

```
MyApp.sln                    ← Solution
├── MyApp.Api/               ← Project 1
│   └── MyApp.Api.csproj
├── MyApp.Common/            ← Project 2
│   └── MyApp.Common.csproj
└── MyApp.Tests/             ← Project 3
    └── MyApp.Tests.csproj
```

---

## Project Structure

| .NET Core | Spring Boot |
|-----------|-------------|
| `Program.cs` | `@SpringBootApplication` main class |
| `Controllers/` | `controller/` package |
| `Services/` | `service/` package |
| `Models/` | `dto/` or `model/` package |
| `appsettings.json` | `application.yml` |
| `*.csproj` | `pom.xml` |

```
MyApp.Api/
├── Controllers/
│   └── ClaimsController.cs
├── Services/
├── Models/
├── Properties/
│   └── launchSettings.json
├── Program.cs
├── appsettings.json
├── appsettings.Development.json
├── Dockerfile
└── MyApp.Api.csproj
```

---

## Controllers

### Spring Boot
```java
@RestController
@RequestMapping("/api/claims")
public class ClaimsController {

    @Autowired
    private ClaimsService claimsService;

    @GetMapping("/{id}")
    public Claim getById(@PathVariable Long id) {
        return claimsService.findById(id);
    }
}
```

### .NET Core
```csharp
[ApiController]
[Route("api/claims")]
public class ClaimsController : ControllerBase
{
    private readonly IClaimsService _claimsService;

    public ClaimsController(IClaimsService claimsService)
    {
        _claimsService = claimsService;
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Claim>> GetById(int id)
    {
        return await _claimsService.FindByIdAsync(id);
    }
}
```

---

## Routes

```csharp
// Explicit route (recommended)
[Route("api/claims")]
public class ClaimsController : ControllerBase

// Token-based route (auto from class name)
[Route("api/[controller]")]   // → api/claims
public class ClaimsController : ControllerBase
```

**Use explicit routes for clarity.**

---

## Return Types

| Return Type | Use Case | Spring Equivalent |
|-------------|----------|-------------------|
| `T` | Always 200 OK | Return object directly |
| `IActionResult` | Control status codes | `ResponseEntity<?>` |
| `ActionResult<T>` | Best of both (recommended) | `ResponseEntity<T>` |

### Common Response Helpers

| .NET Core | HTTP Status | Spring Boot |
|-----------|-------------|-------------|
| `Ok(data)` | 200 | `ResponseEntity.ok(data)` |
| `Created(uri, data)` | 201 | `ResponseEntity.created(uri).body(data)` |
| `NoContent()` | 204 | `ResponseEntity.noContent().build()` |
| `BadRequest()` | 400 | `ResponseEntity.badRequest().build()` |
| `NotFound()` | 404 | `ResponseEntity.notFound().build()` |

### Example
```csharp
[HttpGet("{id}")]
public async Task<ActionResult<Claim>> GetById(int id)
{
    var claim = await _service.FindByIdAsync(id);

    if (claim == null)
        return NotFound();

    return claim;  // Returns 200 OK with JSON
}
```

---

## Async/Await

### Why Use Async?
- Releases thread during I/O operations (DB, HTTP calls)
- Better scalability under load
- Thread is NOT blocked while waiting

### Comparison

| .NET Core | Spring Boot |
|-----------|-------------|
| `async Task<T>` | `Mono<T>` (WebFlux) |
| `await` | `.subscribe()`, `.block()` |
| Easy syntax | Complex reactive chains |

### Syntax
```csharp
// Sync (thread blocked)
public IActionResult GetById(int id)
{
    var claim = _service.FindById(id);  // Thread waits
    return Ok(claim);
}

// Async (thread released)
public async Task<IActionResult> GetById(int id)
{
    var claim = await _service.FindByIdAsync(id);  // Thread free
    return Ok(claim);
}
```

### When to Use Async

| Operation | Use Async? |
|-----------|------------|
| Database calls | Yes |
| HTTP calls | Yes |
| File I/O | Yes |
| In-memory only | No benefit |

### Testing with Delay
```csharp
await Task.Delay(2000);  // Non-blocking 2 sec delay (like async I/O)
Thread.Sleep(2000);      // Blocking 2 sec delay (avoid this)
```

---

## Configuration Files

### File Overview

| File | Purpose | Committed to Git? |
|------|---------|-------------------|
| `appsettings.json` | Base config | Yes |
| `appsettings.Development.json` | Dev overrides | Yes |
| `appsettings.QA.json` | QA overrides | Yes |
| `appsettings.Production.json` | Prod overrides | Yes (without secrets) |
| `launchSettings.json` | Local run config | Optional |
| `secrets.json` | Local secrets | No |

### appsettings.json (like application.yml)
```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information"
    }
  },
  "ConnectionStrings": {
    "Database": "Server=localhost;Database=MyApp"
  }
}
```

### launchSettings.json (run configurations)
```json
{
  "profiles": {
    "http": {
      "commandName": "Project",
      "applicationUrl": "http://localhost:5184",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "qa": {
      "commandName": "Project",
      "applicationUrl": "http://localhost:5184",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "QA"
      }
    }
  }
}
```

### Accessing Config in Code
```csharp
// In constructor (via DI)
public ClaimsController(IConfiguration configuration)
{
    var connString = configuration.GetConnectionString("Database");
    var logLevel = configuration["Logging:LogLevel:Default"];
}
```

---

## Dependencies (NuGet)

### Visual Studio Dependencies Folder

| Folder | Contents | Java Equivalent |
|--------|----------|-----------------|
| **Packages** | NuGet libraries | Maven dependencies |
| **Frameworks** | .NET runtime | JDK modules |
| **Analyzers** | Code analysis | Checkstyle, SonarLint |

### Commands

```bash
# Add package
dotnet add package Swashbuckle.AspNetCore

# Remove package
dotnet remove package Swashbuckle.AspNetCore

# Restore packages
dotnet restore

# List packages
dotnet list package
```

### In .csproj (like pom.xml)
```xml
<ItemGroup>
  <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
</ItemGroup>
```

### NuGet Source
```bash
# Add nuget.org (if missing)
dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org

# List sources
dotnet nuget list source
```

---

## Swagger UI

### Install
```bash
dotnet add package Swashbuckle.AspNetCore
```

### Configure (Program.cs)
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Access
```
http://localhost:5184/swagger
```

---

## Environment-Based Config

### How It Works
```
ASPNETCORE_ENVIRONMENT=QA
     ↓
Loads: appsettings.json + appsettings.QA.json
```

### Config Loading Order (later overrides earlier)
1. `appsettings.json` (base)
2. `appsettings.{Environment}.json` (environment-specific)
3. Environment variables
4. Command line arguments

### Setting Environment

| Context | How to Set |
|---------|------------|
| **Local (VS)** | `launchSettings.json` → `ASPNETCORE_ENVIRONMENT` |
| **CLI** | `dotnet run --environment QA` |
| **Docker** | `ENV ASPNETCORE_ENVIRONMENT=QA` |
| **Kubernetes** | `env` in deployment.yaml |

### Kubernetes Example
```yaml
spec:
  containers:
    - name: claims-api
      image: openpa-claims-api
      env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
```

### Spring Boot Comparison

| .NET Core | Spring Boot |
|-----------|-------------|
| `ASPNETCORE_ENVIRONMENT=QA` | `SPRING_PROFILES_ACTIVE=qa` |
| `appsettings.QA.json` | `application-qa.yml` |
| `--environment QA` | `--spring.profiles.active=qa` |

---

## Common Commands

```bash
# Create new Web API project
dotnet new webapi -n MyApi

# Build
dotnet build

# Run
dotnet run --project MyApi

# Run with specific environment
dotnet run --environment QA

# Run with specific launch profile
dotnet run --launch-profile qa

# Clean
dotnet clean

# Add package
dotnet add package PackageName

# Restore packages
dotnet restore
```

---

## Quick Comparison Table

| Concept | .NET Core | Spring Boot |
|---------|-----------|-------------|
| Entry point | `Program.cs` | `@SpringBootApplication` |
| Controller | `[ApiController]` | `@RestController` |
| Route | `[Route("api/x")]` | `@RequestMapping("/api/x")` |
| GET | `[HttpGet]` | `@GetMapping` |
| POST | `[HttpPost]` | `@PostMapping` |
| Path variable | `[HttpGet("{id}")]` + param | `@PathVariable` |
| Query param | `[FromQuery]` | `@RequestParam` |
| Request body | `[FromBody]` | `@RequestBody` |
| DI | Constructor injection | `@Autowired` |
| Config file | `appsettings.json` | `application.yml` |
| Dependencies | NuGet / `.csproj` | Maven / `pom.xml` |
| Profiles | `ASPNETCORE_ENVIRONMENT` | `SPRING_PROFILES_ACTIVE` |
