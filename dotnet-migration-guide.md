# .NET MVC to .NET Core Microservices Migration Guide

## 1. Migration Strategy

| Approach | When to Use |
|----------|-------------|
| **Strangler Fig** (Recommended) | Migrate one service at a time, route traffic gradually |
| **Big Bang** | Small codebase, tight deadline, high risk |

**Recommendation:** Migrate one bounded context (e.g., Claims) completely, deploy to production, learn from it, then move to the next.

---

## 2. Key Technical Decisions to Make Upfront

| Decision | Options | Recommendation |
|----------|---------|----------------|
| **Solution structure** | Monorepo vs Polyrepo | Monorepo for migration phase |
| **API Gateway** | Custom / YARP / Ocelot / Azure APIM | Azure APIM |
| **Auth** | JWT validation at gateway vs each service | Decide based on security team |
| **Database** | Shared DB vs DB per service | Start shared, split later if needed |
| **Communication** | REST / gRPC / Message Queue | REST for sync, Queue for async |
| **Containerization** | Docker + Kubernetes | Docker + K8s |

---

## 3. What .NET Developers Need to Unlearn

| Old .NET MVC Pattern | New .NET Core Pattern |
|----------------------|----------------------|
| `Global.asax` | `Program.cs` with middleware |
| `Web.config` | `appsettings.json` |
| Unity/Ninject DI | Built-in DI (`builder.Services`) |
| `ApiController` (Web API 2) | `ControllerBase` with `[ApiController]` |
| Full .NET Framework | .NET Core (cross-platform) |
| IIS only | Kestrel, can run anywhere |
| Synchronous code | Async/await everywhere |

---

## 4. .NET Core vs Spring Boot Reference

| .NET Core | Spring Boot Equivalent |
|-----------|------------------------|
| `Program.cs` | `@SpringBootApplication` main class |
| `builder.Services.AddScoped<T>()` | `@Service`, `@Component` beans |
| `appsettings.json` | `application.yml` |
| `IConfiguration` | `@Value`, `@ConfigurationProperties` |
| `DbContext` (EF Core) | `JpaRepository` |
| `ILogger<T>` | `LoggerFactory.getLogger()` |
| `*.csproj` | `pom.xml` |
| NuGet | Maven Central |
| Middleware | Filters / Interceptors |
| `[Authorize]` | `@PreAuthorize` |

---

## 5. Team Structure Recommendation

```
Migration Team
├── .NET Core Experts (2-3)      ← Lead technical patterns
├── Legacy .NET Devs (majority)  ← Domain knowledge, learn Core
├── Tech Lead (Java/Angular)     ← Architecture, code review, standards
└── DevOps (1-2)                 ← CI/CD, Kubernetes, Azure APIM
```

---

## 6. Common Pitfalls to Avoid

| Pitfall | Advice |
|---------|--------|
| **Distributed monolith** | Don't just split code — split by domain boundaries |
| **Shared database coupling** | If services share tables, they're not independent |
| **Sync HTTP chains** | Service A → B → C → D = fragile. Use async messaging |
| **No API versioning** | Add versioning from day one: `/api/v1/claims` |
| **Ignoring async/await** | .NET Core is async-first. Don't block threads. |
| **Over-engineering** | Don't add Kafka, Redis, gRPC on day one. Start simple. |

---

## 7. Migration Checklist Per Service

- [ ] Identify bounded context and models
- [ ] Create new .NET Core Web API project
- [ ] Define API contracts (OpenAPI/Swagger)
- [ ] Migrate models/DTOs
- [ ] Migrate business logic (services)
- [ ] Migrate data access (EF Core or Dapper)
- [ ] Add health checks endpoint
- [ ] Add logging (structured, correlation IDs)
- [ ] Containerize (Dockerfile)
- [ ] CI/CD pipeline
- [ ] Deploy to K8s (staging)
- [ ] Route traffic via Azure APIM
- [ ] Monitor and validate
- [ ] Retire old MVC endpoint

---

## 8. Coding Standards to Enforce

| Standard | Why |
|----------|-----|
| Explicit routes (`[Route("api/claims")]`) | Clarity, no magic |
| Async all the way (`async Task<IActionResult>`) | Performance |
| Dependency injection via constructor | Testability |
| One controller per resource | Maintainability |
| DTOs for API contracts | Don't expose domain entities |
| Correlation ID in all logs | Traceability across services |
| Health endpoints (`/health`) | Kubernetes readiness/liveness |

---

## 9. Project Structure

```
OpenPA.sln
│
├── OpenPA.Claims.Api/              ← Claims Microservice
│   ├── Controllers/
│   │   ├── ClaimsController.cs
│   │   ├── ClaimStatusController.cs
│   │   └── AdjudicationController.cs
│   ├── Services/
│   ├── Repositories/
│   ├── Models/
│   ├── Program.cs
│   ├── appsettings.json
│   └── Dockerfile
│
├── OpenPA.PriorAuth.Api/           ← PriorAuth Microservice
│   └── ...
│
├── OpenPA.Specialty.Api/           ← Specialty Microservice
│   └── ...
│
├── OpenPA.Common/                  ← Shared utilities
│   └── ...
│
└── OpenPA.Domain/                  ← Shared entities
    └── ...
```

---

## 10. Controller Example

### Spring Boot
```java
@RestController
@RequestMapping("/api/claims")
public class ClaimsController {

    private final ClaimsService claimsService;

    @Autowired
    public ClaimsController(ClaimsService claimsService) {
        this.claimsService = claimsService;
    }

    @GetMapping
    public List<Claim> getAll() {
        return claimsService.findAll();
    }

    @GetMapping("/{id}")
    public Claim getById(@PathVariable Long id) {
        return claimsService.findById(id);
    }
}
```

### .NET Core (Equivalent)
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

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        var claims = await _claimsService.FindAllAsync();
        return Ok(claims);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(long id)
    {
        var claim = await _claimsService.FindByIdAsync(id);
        return Ok(claim);
    }
}
```

---

## 11. Quick Wins for Team Buy-In

| Win | Impact |
|-----|--------|
| Swagger UI works out of the box | Devs can test APIs immediately |
| Hot reload (`dotnet watch`) | Faster dev cycle than old .NET |
| Docker runs locally | No "works on my machine" |
| Cross-platform | Devs can use Mac/Linux if they want |

---

## 12. Key Principle

> **Migrate the architecture, not just the code.**

Don't just copy-paste old MVC controllers into new .NET Core projects. Rethink:
- Are these the right service boundaries?
- Should this be async?
- Is this endpoint even needed anymore?

The migration is an opportunity to fix old mistakes, not replicate them.

---

## 13. Architecture with Azure APIM

```
                    ┌─────────────────────────────────────┐
                    │          Azure API Management       │
   Client Request   │  • JWT Validation                   │
   ───────────────► │  • Rate Limiting                    │
                    │  • Routing                          │
                    └─────────────┬───────────────────────┘
                                  │
          ┌───────────────────────┼───────────────────────┐
          ▼                       ▼                       ▼
   ┌─────────────┐        ┌─────────────┐        ┌─────────────┐
   │ Claims.Api  │        │ PriorAuth.  │        │ Specialty.  │
   │             │        │    Api      │        │    Api      │
   └─────────────┘        └─────────────┘        └─────────────┘
        (AKS)                  (AKS)                  (AKS)
```

---

## 14. AOT vs Standard (Decision Reference)

| Aspect | **ASP.NET Core Web API** | **Native AOT** |
|--------|--------------------------|----------------|
| **Compilation** | JIT (Just-In-Time) | AOT (Ahead-Of-Time) |
| **Startup time** | Slower | Much faster |
| **Reflection support** | Full | Limited |
| **Library compatibility** | Full ecosystem | Some libraries don't work |
| **Recommendation** | Use for migration | Consider later for optimization |

**For migration: Use standard ASP.NET Core Web API. Optimize with AOT later if needed.**
