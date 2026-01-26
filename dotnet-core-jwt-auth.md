# JWT Authentication for OpenPA Microservices

## The Narrative

### What We're Doing

We're implementing JWT (JSON Web Token) authentication for our microservices. We'll leverage our existing `validateUser` API — the same logic we use today to validate credentials. The difference is that after validation, we'll generate a JWT token that the client will use for all subsequent API calls.

### How It Works

When a user logs in, they call our `validateUser` endpoint with their username and password. If the credentials are valid, we retrieve their user information — ID, name, email, roles — and package this into a JWT token. We return this token to the client.

For every API call after that, the client includes this token in the request header. Our services automatically validate the token and extract the user information from it. No session, no database lookup on every request — the token contains everything we need.

### Why JWT?

The token is self-contained. Think of it like a passport. When you show your passport at the border, they don't call your government to verify who you are. They trust the passport because it has security features that prove it's authentic. JWT works the same way — the signature proves the token hasn't been tampered with.

This is perfect for microservices because each service can validate the token independently. The Claims API doesn't need to call the Auth service to check if a token is valid. It just verifies the signature using a shared secret key.

---

## Implementation

### Step 1: Install the NuGet Package

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

### Step 2: Add JWT Settings to appsettings.json

```json
{
  "Jwt": {
    "Secret": "YourSecretKeyThatIsAtLeast32CharactersLong!",
    "Issuer": "OpenPA.Auth",
    "Audience": "OpenPA.Services"
  }
}
```

### Step 3: Configure JWT in Program.cs

This is the critical step. We configure the authentication middleware so it automatically validates every incoming request.

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddControllers();

// Configure JWT Authentication
var jwtSecret = builder.Configuration["Jwt:Secret"];
var jwtIssuer = builder.Configuration["Jwt:Issuer"];
var jwtAudience = builder.Configuration["Jwt:Audience"];

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtSecret)),
            ValidateIssuer = true,
            ValidIssuer = jwtIssuer,
            ValidateAudience = true,
            ValidAudience = jwtAudience,
            ValidateLifetime = true
        };
    });

builder.Services.AddAuthorization();

var app = builder.Build();

// Order matters - Authentication must come before Authorization
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
app.Run();
```

### Step 4: Protect Your Controllers

Once configured, protecting endpoints is simple:

```csharp
[ApiController]
[Route("api/claims")]
[Authorize]  // Requires valid JWT
public class ClaimsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok("You are authenticated!");
    }

    [HttpGet("admin")]
    [Authorize(Roles = "Admin")]  // Requires Admin role
    public IActionResult AdminOnly()
    {
        return Ok("You are an admin!");
    }

    [HttpGet("health")]
    [AllowAnonymous]  // No authentication required
    public IActionResult Health()
    {
        return Ok("Healthy");
    }
}
```

### Step 5: Access User Information

The validated token's claims are available through the `User` property:

```csharp
[HttpGet("me")]
[Authorize]
public IActionResult GetCurrentUser()
{
    var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
    var email = User.FindFirst(ClaimTypes.Email)?.Value;
    var roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList();

    return Ok(new { userId, email, roles });
}
```

The `User` property is automatically populated by the authentication middleware from the JWT claims.

---

## The Auth Service (Token Generation)

The Auth service generates tokens when users log in:

```csharp
public class TokenService
{
    private readonly IConfiguration _config;

    public TokenService(IConfiguration config)
    {
        _config = config;
    }

    public string GenerateToken(User user)
    {
        var key = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_config["Jwt:Secret"]));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new List<Claim>
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Email, user.Email),
            new Claim("name", user.FullName)
        };

        foreach (var role in user.Roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        var token = new JwtSecurityToken(
            issuer: _config["Jwt:Issuer"],
            audience: _config["Jwt:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddHours(1),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

---

## Future: Azure Entra ID

When we move to Azure, we'll switch from our custom JWT to Azure Entra ID. The only change is in Program.cs:

```csharp
// Current: Our secret key
IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret))

// Future: Azure Entra ID
options.Authority = "https://login.microsoftonline.com/{tenant-id}";
options.Audience = "{client-id}";
```

The rest of our code — controllers, [Authorize] attributes, User.Claims — stays exactly the same.

---

## Diagram (If They Ask)

```
LOGIN:
  Client → POST /api/auth/validateUser (username, password) → Auth Service
  Client ← { token: "eyJhbG..." } ← Auth Service

SUBSEQUENT CALLS:
  Client → GET /api/claims (Header: Authorization: Bearer eyJhbG...) → Claims API
  Claims API validates token automatically → Returns data
```
