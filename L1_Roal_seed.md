<br>
<br>

# 🔥📝 Way 1 for Roal Seed  📝🔥

<br>
<br>
---

## ✅ Benefits of Extension Method

| ✅ Clean Program.cs         | ✅ Easy Testing                               | ✅ Scalable for User/Role Seeding        |
| -------------------------- | -------------------------------------------- | --------------------------------------- |
| Keeps `Program.cs` minimal | Easily write unit tests or integration tests | Add multiple users, roles, claims, etc. |

---

## ✅ Step-by-Step Implementation Using Extension Method

---

### 🧩 Step 1: Create the Extension Method

📄 `Extensions/ServiceProviderExtensions.cs`

```csharp
using CompliteAuthTestingApp.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.DependencyInjection;

namespace CompliteAuthTestingApp.Extensions
{
    public static class ServiceProviderExtensions
    {
        public static async Task SeedRolesAndAdminAsync(this IServiceProvider serviceProvider)
        {
            // ① Resolve required services
            var roleManager = serviceProvider.GetRequiredService<RoleManager<IdentityRole>>();
            var userManager = serviceProvider.GetRequiredService<UserManager<ApplicationUser>>();

            // ② Define roles
            string[] roles = { "ADMIN", "USER", "STAFF", "MODERATOR" };

            // ③ Create roles if not exist
            foreach (var role in roles)
            {
                if (!await roleManager.RoleExistsAsync(role))
                {
                    await roleManager.CreateAsync(new IdentityRole(role));
                }
            }

            // ④ Create default admin user if not exists
            string adminEmail = "admin@example.com";
            string adminPassword = "Admin@123";

            if (await userManager.FindByEmailAsync(adminEmail) == null)
            {
                var adminUser = new ApplicationUser
                {
                    UserName = adminEmail,
                    Email = adminEmail,
                    EmailConfirmed = true
                };

                var result = await userManager.CreateAsync(adminUser, adminPassword);

                if (result.Succeeded)
                {
                    await userManager.AddToRoleAsync(adminUser, "ADMIN");
                }
            }
        }
    }
}
```

---

### 🛠️ Step 2: Call the Extension from `Program.cs`

📄 `Program.cs`

```csharp
using CompliteAuthTestingApp.Models;
using CompliteAuthTestingApp.Data;
using CompliteAuthTestingApp.Extensions; // 👈 import your extension
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Register DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Register Identity
builder.Services.AddIdentity<ApplicationUser, IdentityRole>()
    .AddEntityFrameworkStores<AppDbContext>()
    .AddDefaultTokenProviders();

// Register other services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// 👇 Clean and concise call to your extension method
using (var scope = app.Services.CreateScope())
{
    await scope.ServiceProvider.SeedRolesAndAdminAsync();
}

app.UseSwagger();
app.UseSwaggerUI();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

---

###  Summary

| File                           | Responsibility                            |
| ------------------------------ | ----------------------------------------- |
| `ServiceProviderExtensions.cs` | Seeding logic for roles & admin           |
| `Program.cs`                   | Clean startup logic (calls the extension) |

---
<br>
<br>

# 🔥📝 Way 2 for Roal Seed  📝🔥

<br>
<br>





## ✅ Step 1: Create `SeedRole.cs` (for roles + admin user)

📄 `Data/SeedRole.cs`

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.DependencyInjection;
using CompliteAuthTestingApp.Models;

namespace CompliteAuthTestingApp.Data
{
    public static class SeedRole
    {
        public static async Task InitializeAsync(IServiceProvider serviceProvider)
        {
            var roleManager = serviceProvider.GetRequiredService<RoleManager<IdentityRole>>();
            var userManager = serviceProvider.GetRequiredService<UserManager<ApplicationUser>>();

            string[] roles = { "ADMIN", "USER", "STAFF", "MODERATOR" };

            // ✅ 1. Seed Roles
            foreach (var role in roles)
            {
                if (!await roleManager.RoleExistsAsync(role))
                {
                    await roleManager.CreateAsync(new IdentityRole(role));
                }
            }

            // ✅ 2. Seed Default Admin User
            var adminEmail = "admin@example.com";
            var adminPassword = "Admin@123"; // 🔐 You can later hash or read from secrets

            if (await userManager.FindByEmailAsync(adminEmail) == null)
            {
                var adminUser = new ApplicationUser
                {
                    UserName = adminEmail,
                    Email = adminEmail,
                    EmailConfirmed = true
                };

                var result = await userManager.CreateAsync(adminUser, adminPassword);
                if (result.Succeeded)
                {
                    await userManager.AddToRoleAsync(adminUser, "ADMIN");
                }
            }
        }
    }
}
```

---

## ✅ Step 2: Modify `Program.cs` to Call the Seeder

📄 `Program.cs`

```csharp
using CompliteAuthTestingApp.Data;
using CompliteAuthTestingApp.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// ✅ Add DbContext
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// ✅ Add Identity
builder.Services.AddIdentity<ApplicationUser, IdentityRole>()
    .AddEntityFrameworkStores<AppDbContext>()
    .AddDefaultTokenProviders();

// ✅ Add other services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// ✅ Seed roles + admin user
using (var scope = app.Services.CreateScope())
{
    var services = scope.ServiceProvider;
    await SeedRole.InitializeAsync(services);
}

app.UseSwagger();
app.UseSwaggerUI();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

---

## ✅ Step 3: Ensure `ApplicationUser.cs` Exists

📄 `Models/ApplicationUser.cs`

```csharp
using Microsoft.AspNetCore.Identity;

namespace CompliteAuthTestingApp.Models
{
    public class ApplicationUser : IdentityUser
    {
        // Add custom properties if needed
    }
}
```

---

## ✅ Step 4: Run the Migrations and Database Update

```bash
dotnet ef migrations add InitIdentity
dotnet ef database update
dotnet run
```

---

## 🎉 Result

* Roles (`ADMIN`, `USER`, `STAFF`, `MODERATOR`) are created.
* Default admin user `admin@example.com` with password `Admin@123` is created and added to `ADMIN` role.

---

Let me know if you also want to:

* Automatically **login the admin on startup** for debugging
* **Change password storage**
* Or **restrict controllers by role** (`[Authorize(Roles = "ADMIN")]`)