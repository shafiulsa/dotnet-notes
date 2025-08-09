* add **JWT token generation** using your existing **Identity** setup,
* add a **clean extension method** to register JWT + Swagger,
* add a **TokenService**, and
* add a **login endpoint** that returns a JWT (your `register` stays as-is).

Follow these steps **in order**.

---

# 1) Add required NuGet packages

From your project folder run:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Swashbuckle.AspNetCore
```

(You already have Identity & EF packages — these two are for JWT bearer validation and Swagger.)

---

# 2) Add JWT settings to `appsettings.json`

Add a `Jwt` section (keep secrets out of source control in production):

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=MyApiDb;Trusted_Connection=True;TrustServerCertificate=True"
  },
  "Jwt": {
    "Key": "super_secret_long_key_here_please_change!",   // ✅ replace for production
    "Issuer": "CompliteAuthTestingApp",
    "Audience": "CompliteAuthTestingAppClients",
    "DurationInMinutes": "60"
  },
  "Logging": { "LogLevel": { "Default": "Information", "Microsoft": "Warning" } },
  "AllowedHosts": "*"
}
```

---

# 3) Create `Services/TokenService.cs`

Create folder `Services/` and file `TokenService.cs`. This builds the JWT from an `ApplicationUser` and their roles.

```csharp
// Services/TokenService.cs
using System;
using System.Collections.Generic;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using CompliteAuthTestingApp.Models;
using Microsoft.Extensions.Configuration;
using Microsoft.IdentityModel.Tokens;

namespace CompliteAuthTestingApp.Services
{
    // ✅ Simple token builder used by AuthController
    public class TokenService
    {
        private readonly IConfiguration _configuration;
        public TokenService(IConfiguration configuration)
        {
            _configuration = configuration;
        }

        // CreateToken: returns a signed JWT string that includes role claims
        public string CreateToken(ApplicationUser user, IList<string> roles)
        {
            // 1) Basic claims
            var claims = new List<Claim>
            {
                new Claim(ClaimTypes.NameIdentifier, user.Id ?? string.Empty),
                new Claim(ClaimTypes.Name, user.UserName ?? string.Empty),
                new Claim(JwtRegisteredClaimNames.Sub, user.UserName ?? string.Empty),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
            };

            // 2) Add role claims
            foreach (var role in roles)
            {
                claims.Add(new Claim(ClaimTypes.Role, role));
            }

            // 3) Create signing key & credentials
            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

            // 4) Expiration
            var duration = int.TryParse(_configuration["Jwt:DurationInMinutes"], out var m) ? m : 60;
            var expires = DateTime.UtcNow.AddMinutes(duration);

            // 5) Create token
            var jwt = new JwtSecurityToken(
                issuer: _configuration["Jwt:Issuer"],
                audience: _configuration["Jwt:Audience"],
                claims: claims,
                expires: expires,
                signingCredentials: creds
            );

            return new JwtSecurityTokenHandler().WriteToken(jwt);
        }
    }
}
```

---

# 4) Create an extension for registering JWT + Swagger

Create folder `Extensions/` and file `ServiceExtensions.cs`. This keeps `Program.cs` clean.

```csharp
// Extensions/ServiceExtensions.cs
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.IdentityModel.Tokens;
using Microsoft.OpenApi.Models;
using CompliteAuthTestingApp.Services;

namespace CompliteAuthTestingApp.Extensions
{
    public static class ServiceExtensions
    {
        // ✅ Registers TokenService, JWT authentication and Swagger with Bearer support
        public static IServiceCollection AddJwtAndSwagger(this IServiceCollection services, IConfiguration configuration)
        {
            // Register token service that will build the JWTs
            services.AddScoped<TokenService>();

            // read jwt settings
            var jwtKey = configuration["Jwt:Key"];
            var jwtIssuer = configuration["Jwt:Issuer"];
            var jwtAudience = configuration["Jwt:Audience"];

            // Add Authentication & JWT Bearer
            services.AddAuthentication(options =>
            {
                options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
                options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
            })
            .AddJwtBearer(options =>
            {
                // In production, RequireHttpsMetadata should be true
                options.RequireHttpsMetadata = true;
                options.SaveToken = true;
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidateAudience = true,
                    ValidateLifetime = true,
                    ValidateIssuerSigningKey = true,
                    ValidIssuer = jwtIssuer,
                    ValidAudience = jwtAudience,
                    IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtKey))
                };
            });

            // Swagger with Bearer support
            services.AddEndpointsApiExplorer();
            services.AddSwaggerGen(c =>
            {
                c.SwaggerDoc("v1", new OpenApiInfo { Title = "CompliteAuthTestingApp API", Version = "v1" });

                var securityScheme = new OpenApiSecurityScheme
                {
                    Name = "Authorization",
                    Description = "Enter JWT token only (no 'Bearer ' prefix). Example: eyJhbGciOi...",
                    In = ParameterLocation.Header,
                    Type = SecuritySchemeType.Http,
                    Scheme = "bearer",
                    BearerFormat = "JWT"
                };

                c.AddSecurityDefinition("Bearer", securityScheme);

                c.AddSecurityRequirement(new OpenApiSecurityRequirement
                {
                    {
                        new OpenApiSecurityScheme {
                            Reference = new OpenApiReference {
                                Type = ReferenceType.SecurityScheme,
                                Id = "Bearer"
                            }
                        },
                        new string[] {}
                    }
                });
            });

            return services;
        }
    }
}
```

---

# 5) Update `Program.cs`

Replace the `AddEndpointsApiExplorer()` / `AddSwaggerGen()` block in your `Program.cs` with a call to the extension method and ensure authentication middleware is added.

Here’s a cleaned up `Program.cs` (integrates your existing Identity registration + the new extension):

```csharp
using CompliteAuthTestingApp.Data;
using CompliteAuthTestingApp.Extensions;
using CompliteAuthTestingApp.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// 1) DB context (your existing line)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// 2) Identity registration (keep your existing config)
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.Password.RequireNonAlphanumeric = false;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();

// 3) ✅Add JWT + Swagger + TokenService (extension method)
builder.Services.AddJwtAndSwagger(builder.Configuration);

// 4) Add controllers
builder.Services.AddControllers();

var app = builder.Build();

// Seed roles and admin (your existing code)
using (var scope = app.Services.CreateScope())
{
    await scope.ServiceProvider.SeedRolesAndAdminAsync();
}

// 5) Middlewares
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();

// IMPORTANT: Authentication must come before Authorization & before endpoints
app.UseAuthentication();   // ✅ validates token and builds HttpContext.User
app.UseAuthorization();

app.MapControllers();

app.Run();
```

> **Note:** We kept your Identity configuration in `Program.cs` unchanged; the extension only adds JWT auth, TokenService and Swagger.

---

# 6) Update `AuthController` — add Login endpoint that issues JWT

Edit `Controllers/AuthController.cs`. We will **inject `SignInManager` and `TokenService`** and add the `login` action.

Replace your existing file content with:

```csharp
using System.Threading.Tasks;
using CompliteAuthTestingApp.Models;
using CompliteAuthTestingApp.Services;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;

namespace CompliteAuthTestingApp.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class AuthController : ControllerBase
    {
        private readonly UserManager<ApplicationUser> _userManager;
        private readonly RoleManager<IdentityRole> _roleManager;
        private readonly SignInManager<ApplicationUser> _signInManager;
        private readonly TokenService _tokenService;

        // Updated constructor: SignInManager + TokenService injected
        public AuthController(
            UserManager<ApplicationUser> userManager,
            RoleManager<IdentityRole> roleManager,
            SignInManager<ApplicationUser> signInManager,
            TokenService tokenService)
        {
            _userManager = userManager;
            _roleManager = roleManager;
            _signInManager = signInManager;
            _tokenService = tokenService;
        }

        // Register user (your existing method)
        [HttpPost("register")]
        public async Task<IActionResult> Register(RegisterDto model)
        {
            var user = new ApplicationUser
            {
                UserName = model.Email,
                Email = model.Email,
                FullName = model.FullName,
                StoreId = model.StoreId,
                IsApproved = true
            };
            var result = await _userManager.CreateAsync(user, model.Password);
            if (!result.Succeeded)
            {
                return BadRequest(result.Errors);
            }

            await _userManager.AddToRoleAsync(user, "User"); // default role User
            return Ok("User registered successfully");
        }

        // ✅ Login endpoint — validates credentials via Identity and returns JWT
        [HttpPost("login")]
        public async Task<IActionResult> Login([FromBody] LoginDto model)
        {
            if (model == null) return BadRequest("Invalid client request");

            // 1) Find user by email
            var user = await _userManager.FindByEmailAsync(model.Email);
            if (user == null) return Unauthorized(new { message = "Invalid credentials" });

            // 2) Check password (does not sign-in to cookie, just verifies)
            var passwordCheck = await _signInManager.CheckPasswordSignInAsync(user, model.Password, lockoutOnFailure: false);
            if (!passwordCheck.Succeeded) return Unauthorized(new { message = "Invalid credentials" });

            // 3) Get user roles (so we can include them in token)
            var roles = await _userManager.GetRolesAsync(user);

            // 4) Create token
            var token = _tokenService.CreateToken(user, roles);

            // 5) Return token
            return Ok(new { token });
        }
    }
}
```

**Comments:**

* We use `CheckPasswordSignInAsync` to verify credentials without cookie signin (suitable for API + JWT).
* Token includes role claims (so `[Authorize(Roles="Admin")]` works).

---

# 7) Small fixes & notes for `SeedRole` (optional)

Your `SeedRole` uses `serviceProvider.GetRequiredService<...>()`. If you see a compile error, add:

```csharp
using Microsoft.Extensions.DependencyInjection;
```

at top of `SeedRole` file.

---

# 8) Run EF migrations (if not already)

If you haven’t created the DB schema for Identity tables:

```bash
dotnet ef migrations add InitIdentity
dotnet ef database update
```

(You already call `SeedRolesAndAdminAsync()` in `Program.cs`, so roles/admin will be created at startup.)

---



# 10) Where to put files (final folder view)

```
CompliteAuthTestingApp/
├─ Controllers/
│  └─ AuthController.cs          // updated login + register
├─ Data/
│  └─ AppDbContext.cs
├─ Extensions/
│  └─ ServiceExtensions.cs       // AddJwtAndSwagger extension
├─ Models/
│  └─ ApplicationUser.cs
│  └─ LoginDto.cs
│  └─ RegisterDto.cs
├─ Services/
│  └─ TokenService.cs
├─ Program.cs
├─ appsettings.json
└─ ...
```

---

## Extra suggestions (quick)

* **Production secrets:** use user-secrets / env vars / Key Vault for `Jwt:Key`.
* **Short tokens + refresh tokens**: consider implementing refresh tokens if you need long sessions.
* **HTTPS:** keep `RequireHttpsMetadata = true` in production.
* **Token lifetime & blacklisting:** When changing roles/revoking a user, tokens remain valid until expiry; plan refresh/blacklist if needed.

---