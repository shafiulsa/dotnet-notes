

---

# **Masterpiece Note: VS Code 🐞 Debugger Setup for ASP.NET Core on Ubuntu**

## **1️ Project Folder Structure**

```
My_E_Comm/
│
├─ Controllers/
│   ├─ AuthController.cs
│   └─ MathController.cs       # Example: sum endpoint
│
├─ Data/
│   └─ AppDbContext.cs
│
├─ Models/
│   ├─ ApplicationUser.cs
│   └─ RegisterModel.cs
│
├─ Properties/
│   └─ launchSettings.json
│
├─ Program.cs
├─ My_E_Comm.csproj
│
├─ .vscode/
│   ├─ launch.json
│   └─ tasks.json
│
└─ README.md
```

> **Note:** Keep all files properly named; VS Code will use `.vscode/launch.json` & `.vscode/tasks.json` for debugging.

---

## **2️Program.cs**

This is your main entry point. It includes **Swagger, Identity, EF Core, CORS, HTTPS redirection**, and **debug-friendly settings**.

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using My_E_Comm.Data;
using My_E_Comm.Models;

var builder = WebApplication.CreateBuilder(args);

// =======================
// Add services to DI container
// =======================

// Add controllers and API explorer
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Configure EF Core with SQL Server
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Configure Identity
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = false;
    options.Password.RequiredLength = 6;
})
    .AddEntityFrameworkStores<AppDbContext>()
    .AddDefaultTokenProviders();

// Configure CORS for development
// Allows calls from Swagger, Angular, React, Postman, Thunder Client
builder.Services.AddCors(options =>
{
    options.AddPolicy("DevCors", policy =>
    {
        policy.WithOrigins(
                "http://localhost:5045", // Swagger
                "http://localhost:4200", // Angular dev server
                "http://localhost:3000"  // React dev server
            )
            .AllowAnyMethod()
            .AllowAnyHeader();
    });
});

// =======================
// Build the app
// =======================
var app = builder.Build();

// =======================
// Middleware
// =======================

// Swagger UI available only in Development
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

// Redirect HTTP requests to HTTPS
app.UseHttpsRedirection();

// Apply CORS policy BEFORE authentication/authorization
app.UseCors("DevCors");

// Authentication & Authorization
app.UseAuthentication();
app.UseAuthorization();

// Map controllers (API endpoints)
app.MapControllers();

// Run the app
app.Run();
```

> **Tip:** Always apply CORS before auth middleware, otherwise frontend requests may fail.

---

## **3️ launchSettings.json**

This file configures **the URL the API listens on** and **launch browser settings**.

```json
{
  "$schema": "http://json.schemastore.org/launchsettings.json",
  "profiles": {
    "My_E_Comm": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",         // Opens Swagger automatically
      "applicationUrl": "http://localhost:5045",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  }
}
```

> **Important:** Use **HTTP in development** to avoid SSL trust issues on Ubuntu.

---

## **4️ VS Code Debugger Config: launch.json**

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": ".NET Core Launch",
      "type": "coreclr",
      "request": "launch",
      "preLaunchTask": "build", // Build project before running
      "program": "${workspaceFolder}/My_E_Comm/bin/Debug/net8.0/My_E_Comm.dll",
      "args": [],
      "cwd": "${workspaceFolder}/My_E_Comm",
      "stopAtEntry": false,
      "console": "integratedTerminal",
      "serverReadyAction": {
        "action": "openExternally",            // Opens Swagger automatically
        "pattern": "\\bNow listening on:\\s+(https?://\\S+)"
      },
      "env": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  ]
}
```

> **How it works:**
>
> * Detects when API is ready (`Now listening on:`)
> * Opens Swagger automatically
> * Breakpoints in controllers hit when API is called

---

## **5️ VS Code Build Task: tasks.json**

```json
{
  "version": "2.0.0",
  "tasks": [
    {
      "label": "build",
      "command": "dotnet",
      "type": "process",
      "args": [
        "build",
        "${workspaceFolder}/My_E_Comm/My_E_Comm.csproj",
        "/property:GenerateFullPaths=true",
        "/consoleloggerparameters:NoSummary"
      ],
      "problemMatcher": "$msCompile"
    }
  ]
}
```

> **Purpose:** Builds the project automatically before starting the debugger.

---

## **6️ Example Controller: MathController**

```csharp
using Microsoft.AspNetCore.Mvc;

namespace My_E_Comm.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class MathController : ControllerBase
    {
        // GET api/Math/sum?num1=5&num2=10
        [HttpGet("sum")]
        public IActionResult Sum([FromQuery] int num1, [FromQuery] int num2)
        {
            int result = num1 + num2;
            return Ok(new { Num1 = num1, Num2 = num2, Sum = result });
        }
    }
}
```

* Works in **Swagger, Postman, Thunder Client, Angular, React**.
* Example request:

```
GET http://localhost:5045/api/Math/sum?num1=7&num2=3
```

* Response:

```json
{
  "Num1": 7,
  "Num2": 3,
  "Sum": 10
}
```

---

## **7️ Testing Flow**

| Tool                        | URL / Request                                                                        | Notes                         |
| --------------------------- | ------------------------------------------------------------------------------------ | ----------------------------- |
| **Swagger**                 | [http://localhost:5045/swagger/index.html](http://localhost:5045/swagger/index.html) | Breakpoints hit automatically |
| **Postman / ThunderClient** | [http://localhost:5045/api/](http://localhost:5045/api/)...                          | Works with HTTP/DevCors       |
| **Angular**                 | fetch/axios [http://localhost:4200/api/](http://localhost:4200/api/)...              | DevCors allows calls          |
| **React**                   | fetch/axios [http://localhost:3000/api/](http://localhost:3000/api/)...              | DevCors allows calls          |
| **VS Code Debugger**        | F5 → .NET Core Launch                                                                | Breakpoints hit               |

---

## **8️ Key Notes / Best Practices**

1. **CORS Policy:** Always allow local development origins (`Swagger, Angular, React`).
2. **HTTPS vs HTTP:** On Ubuntu, dev certificates often fail; prefer HTTP in dev.
3. **Breakpoints:** Only work if **API is running in debug mode** via VS Code (`launch.json`).
4. **Swagger:** Opens automatically using `serverReadyAction`.
5. **Frontend Integration:** React/Angular can call API seamlessly with CORS enabled.

---




## **1️💡 How breakpoints work in .NET Core / VS Code**

* Breakpoints are **only hit when the application is running in debug mode**.
* This means **VS Code debugger must be attached** to your API.

> If the API is just running via `dotnet run` or from another terminal **without the debugger attached**, breakpoints will **not hit**.

---

## **2️ Swagger vs Frontend calls**

* **Swagger:** When you press F5 in VS Code with your launch configuration, Swagger opens automatically. Any API call from Swagger hits breakpoints. ☑
* **Frontend (Angular/React):**

  * If your API is running in **debug mode** (VS Code F5), any API request from the frontend (clicking submit, fetching data) will hit the breakpoints. ☑
  * If your API is **not running in debug mode**, breakpoints will **not trigger**, even if the frontend calls the API. ❌

---

## **3️ Summary Table**

| Scenario              | API running in debug mode? | Breakpoints hit? |
| --------------------- | -------------------------- | ---------------- |
| Swagger call          | ☑                          | ☑                |
| Angular/React call    | ☑                          | ☑                |
| Angular/React call    | ❌                          | ❌                |
| Postman/ThunderClient | ☑                          | ☑                |
| Postman/ThunderClient | ❌                          | ❌                |

> **Key point:** Breakpoints always need the debugger attached. The client (Swagger, Postman, Angular, React) doesn’t matter — it’s the **server debug session** that controls breakpoint triggering.

---

## **4️ Recommended workflow for frontend testing**

1. **Start API in VS Code debug mode** (F5 → `.NET Core Launch`).
2. **Open frontend** (Angular or React dev server).
3. Click buttons / send API requests.
4. Breakpoints in your controllers will trigger **as expected**.

> You don’t need Swagger if your frontend is working — but you **must run API in debug mode**.
