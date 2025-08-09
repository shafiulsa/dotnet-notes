

# Step-by-step complete setup

---

### 1. Create the project

```bash
dotnet new webapi -n CustomIdentityApi
cd CustomIdentityApi
```

---

### 2. Add NuGet packages

```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
```

---

### 3. Folder Structure

```
CustomIdentityApi/
│
├── Controllers/
│   └── AuthController.cs       # API endpoints for auth
│
├── Data/
│   └── ApplicationDbContext.cs # EF Core DbContext with Identity
│
├── Models/
│   ├── ApplicationUser.cs      # Custom Identity User
│   ├── RegisterModel.cs        # DTO for register
│   └── LoginModel.cs           # DTO for login
│
├── Program.cs                  # App setup
├── appsettings.json            # Configurations
├── CustomIdentityApi.csproj
```

---

### 4. Add custom user model

Create `Models/ApplicationUser.cs`:

```csharp
using Microsoft.AspNetCore.Identity;

namespace CustomIdentityApi.Models
{
    // Custom user class inherits from IdentityUser
    public class ApplicationUser : IdentityUser
    {
        // Add extra properties as needed
        public string FullName { get; set; } = string.Empty;
        public DateTime DateOfBirth { get; set; }
    }
}
```

---

### 5. Add DTO models

Create `Models/RegisterModel.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace CustomIdentityApi.Models
{
    public class RegisterModel
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; } = string.Empty;

        [Required]
        [MinLength(6, ErrorMessage = "Password must be at least 6 characters")]
        public string Password { get; set; } = string.Empty;

        [Required]
        public string FullName { get; set; } = string.Empty;

        [Required]
        public DateTime DateOfBirth { get; set; }
    }
}
```

Create `Models/LoginModel.cs`:

```csharp
using System.ComponentModel.DataAnnotations;

namespace CustomIdentityApi.Models
{
    public class LoginModel
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; } = string.Empty;

        [Required]
        public string Password { get; set; } = string.Empty;
    }
}
```

---

### 6. Create DbContext

Create `Data/ApplicationDbContext.cs`:

```csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using CustomIdentityApi.Models;

namespace CustomIdentityApi.Data
{
    // DbContext extending IdentityDbContext with custom user
    public class ApplicationDbContext : IdentityDbContext<ApplicationUser>
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options)
        {
        }

        // Add your other DbSets here if needed
    }
}
```

---

### 7. Configure `Program.cs`

Replace existing content in `Program.cs` with this:

```csharp
using CustomIdentityApi.Data;
using CustomIdentityApi.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// Configure SQL Server connection string (update for your setup)
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection") 
    ?? "Server=localhost;Database=CustomIdentityDb;User Id=sa;Password=YourStrong!Passw0rd;TrustServerCertificate=True;";

// Add DbContext with SQL Server provider
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));

// Add Identity services and configure options
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = false;
    options.Password.RequiredLength = 6;
})
    .AddEntityFrameworkStores<ApplicationDbContext>()
    .AddDefaultTokenProviders();

// Add controllers
builder.Services.AddControllers();

// Configure authentication cookie (optional for Web API, usually JWT is preferred for APIs)
// But since no JWT, we still enable authentication middleware
builder.Services.AddAuthentication(IdentityConstants.ApplicationScheme);

var app = builder.Build();

// Middleware pipeline
app.UseHttpsRedirection();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

---

### 8. Add connection string in `appsettings.json`

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=CustomIdentityDb;User Id=sa;Password=YourStrong!Passw0rd;TrustServerCertificate=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

---

### 9. Create Migration and update DB

Make sure dotnet-ef tool installed:

```bash
dotnet tool install --global dotnet-ef
```

Then:

```bash
dotnet ef migrations add InitialIdentityMigration
dotnet ef database update
```

This creates all Identity tables with your custom ApplicationUser columns.

---

### 10. Implement AuthController with Register and Login

Create `Controllers/AuthController.cs`:

```csharp
using CustomIdentityApi.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;

namespace CustomIdentityApi.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class AuthController : ControllerBase
    {
        private readonly UserManager<ApplicationUser> _userManager;
        private readonly SignInManager<ApplicationUser> _signInManager;

        public AuthController(UserManager<ApplicationUser> userManager, SignInManager<ApplicationUser> signInManager)
        {
            _userManager = userManager;
            _signInManager = signInManager;
        }

        // POST: api/auth/register
        [HttpPost("register")]
        public async Task<IActionResult> Register([FromBody] RegisterModel model)
        {
            if (!ModelState.IsValid)
                return BadRequest(ModelState);

            // Check if user already exists
            var userExists = await _userManager.FindByEmailAsync(model.Email);
            if (userExists != null)
                return Conflict("User already exists with this email.");

            // Create new ApplicationUser instance
            var user = new ApplicationUser
            {
                UserName = model.Email,
                Email = model.Email,
                FullName = model.FullName,
                DateOfBirth = model.DateOfBirth
            };

            // Create user with password
            var result = await _userManager.CreateAsync(user, model.Password);

            if (!result.Succeeded)
            {
                // Return all errors
                var errors = result.Errors.Select(e => e.Description);
                return BadRequest(new { Errors = errors });
            }

            return Ok("User registered successfully.");
        }

        // POST: api/auth/login
        [HttpPost("login")]
        public async Task<IActionResult> Login([FromBody] LoginModel model)
        {
            if (!ModelState.IsValid)
                return BadRequest(ModelState);

            var user = await _userManager.FindByEmailAsync(model.Email);
            if (user == null)
                return Unauthorized("Invalid email or password.");

            // Check password sign-in
            var result = await _signInManager.CheckPasswordSignInAsync(user, model.Password, false);

            if (!result.Succeeded)
                return Unauthorized("Invalid email or password.");

            // Normally for Web API you would return JWT token here.
            // Since no JWT, we just return success message for demo.

            return Ok("User logged in successfully.");
        }
    }
}
```

---

### 11. How to test

Use Postman or curl to test:

* **Register**

POST `https://localhost:5001/api/auth/register`
Body (JSON):

```json
{
  "email": "user@example.com",
  "password": "Password123",
  "fullName": "John Doe",
  "dateOfBirth": "1990-01-01"
}
```

* **Login**

POST `https://localhost:5001/api/auth/login`
Body (JSON):

```json
{
  "email": "user@example.com",
  "password": "Password123"
}
```

---

# Summary

* Created custom ApplicationUser with extra fields
* Set up EF Core Identity without JWT
* Implemented Register and Login endpoints
* Provided full folder structure
* Detailed comments in code

---

<br>
<br>
<br>
<br>
<br>
<br>

## 🛠️ Note 
Here’s a full list of the **common options** you can configure in

```csharp
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    // ----- Password Settings -----
    options.Password.RequireDigit = true;                  // Require at least one digit
    options.Password.RequireLowercase = true;              // Require at least one lowercase letter
    options.Password.RequireUppercase = true;              // Require at least one uppercase letter
    options.Password.RequireNonAlphanumeric = true;        // Require a non-alphanumeric character (e.g., @, #, !)
    options.Password.RequiredLength = 6;                   // Minimum password length
    options.Password.RequiredUniqueChars = 1;              // Minimum unique characters in the password

    // ----- Lockout Settings -----
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(5);  // Lockout time
    options.Lockout.MaxFailedAccessAttempts = 5;                       // Max failed attempts before lockout
    options.Lockout.AllowedForNewUsers = true;                         // Lockout new users?

    // ----- User Settings -----
    options.User.RequireUniqueEmail = true;              // Require each user to have a unique email
    options.User.AllowedUserNameCharacters =              // Allowed characters in the username
        "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789-._@+";

    // ----- Sign-in Settings -----
    options.SignIn.RequireConfirmedEmail = false;         // Require email confirmation before login
    options.SignIn.RequireConfirmedPhoneNumber = false;   // Require phone number confirmation before login
    options.SignIn.RequireConfirmedAccount = false;       // Require account confirmation before login
});
```

### **Categories in Detail**

1. **Password**

   * Controls complexity requirements for user passwords.
2. **Lockout**

   * Controls failed login attempts before temporarily locking an account.
3. **User**

   * Controls username restrictions and uniqueness of email.
4. **SignIn**

   * Controls confirmation requirements before signing in.

---


# 👉CRUD Operations and Their Possible Subcategories / Subcases

### 1. **Create**

* **Create Single**: Add one new record/item/entity.
* **Create Bulk** (Batch Create): Add multiple records/items at once.
* **Create with Validation**: Create with data validation rules applied.
* **Create with Default Values**: Create using defaults for some fields.
* **Create with Relations**: Create an entity along with related child entities.
* **Create with File Upload**: Create with attached files or images.
* **Create Draft**: Save a record as draft before final submission.
* **Create and Return ID**: Create and return the unique identifier or full object.
* **Create with Conditional Logic**: Create depending on certain business rules.

---

### 2. **Read**

* **Read Single**: Get details of one specific record/item by ID or unique key.
* **Read Multiple**: Fetch a list or collection of records.
* **Read All**: Fetch all records without filtering (careful with large datasets).
* **Read with Filters**: Get records filtered by specific fields or conditions.
* **Read with Pagination**: Return records in pages (e.g., 10 per page).
* **Read with Sorting**: Return records sorted by one or more fields.
* **Read with Search**: Search records by keywords or patterns.
* **Read with Projection**: Fetch only specific fields instead of whole records.
* **Read with Related Data**: Fetch an entity plus its related entities (e.g., join or include).
* **Read Metadata / Summary**: Fetch metadata or summary stats about the data.
* **Read Soft-Deleted Records**: Fetch records marked as deleted but not physically removed.

---

### 3. **Update**

* **Update Single**: Modify one record by ID.
* **Update Bulk**: Modify multiple records at once.
* **Update Partial** (Patch): Modify only specific fields, not whole record.
* **Update Full** (Put): Replace the whole record with new data.
* **Update with Validation**: Apply validation rules before updating.
* **Update with Audit Trail**: Track who and when updated the record.
* **Update Status/Flag Only**: Update just a status or flag field (e.g., active/inactive).
* **Update with Conflict Handling**: Manage concurrent updates (optimistic locking).
* **Update and Return Updated Object**: Return the modified entity after update.

---

### 4. **Delete**

* **Delete Single**: Remove one record by ID.
* **Delete Bulk**: Remove multiple records at once.
* **Soft Delete**: Mark record as deleted without physical removal.
* **Hard Delete**: Permanently remove the record.
* **Delete with Confirmation**: Require confirmation before deletion.
* **Delete with Dependency Check**: Prevent deletion if related records exist.
* **Delete Cascade**: Automatically delete related child records.
* **Delete and Archive**: Move deleted records to archive storage.
* **Delete with Audit Trail**: Log delete actions for traceability.

---

### Bonus: Additional Actions Often Related to CRUD

* **Restore**: Undo soft delete by restoring a deleted record.
* **Upsert**: Create or update in one action depending on existence.
* **Clone/Duplicate**: Copy an existing record to create a new one.
* **Export**: Export data (read operation) to CSV, Excel, PDF, etc.
* **Import**: Import data in bulk to create or update records.

---
# Disable

**Disable** is often considered a **soft state change** rather than a full delete or update, but it’s very common in APIs and apps, especially for things like users, products, features, or accounts.

---

### What is a **Disable** endpoint?

* It marks an entity as inactive, disabled, or unavailable without deleting it.
* The entity remains in the database but is excluded from active lists or prevented from performing certain actions.
* It is often implemented as a **soft delete** or a **status flag update** (e.g., setting `isActive = false` or `status = "disabled"`).

---

### How Disable fits in CRUD?

| Operation | Typical Method | Description                                       |
| --------- | -------------- | ------------------------------------------------- |
| Disable   | PATCH / PUT    | Update the record’s status to disabled/inactive   |
| Enable    | PATCH / PUT    | Update the record’s status back to enabled/active |

---

### Disable Endpoint Subcases

* **Disable Single**: Disable one specific item by ID.
* **Disable Bulk**: Disable multiple items at once (batch).
* **Disable with Reason**: Optionally provide a reason for disabling.
* **Disable with Audit Trail**: Log who disabled the record and when.
* **Disable with Notification**: Notify relevant users or systems when disabled.
* **Soft Disable**: Just flag as disabled without physical deletion.
* **Conditional Disable**: Only disable if certain conditions or dependencies are met.

---

### Example API endpoints naming for Disable:

* `PATCH /items/{id}/disable` — disable a single item
* `PATCH /items/disable` — disable multiple items by sending an array of IDs
* `PATCH /users/{id}/disable` — disable a user account
* `PATCH /features/{id}/toggle` — enable/disable toggle

---

### Example JSON payload to disable with reason:

```json
{
  "isActive": false,
  "reason": "Outdated product, discontinued"
}
```

---

### Summary:

* **Disable** is a specialized **Update** operation, usually a status change.
* It allows reversible deactivation without data loss.
* Very useful in real-world apps for safety and auditability.

---

# Code impliment


### Example Entity

```csharp
public class Item
{
    public int Id { get; set; }
    public string Name { get; set; }
    public bool IsActive { get; set; } = true; // true means enabled, false means disabled
}
```

---

### Example DbContext (EF Core)

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Item> Items { get; set; }

    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }
}
```

---

### Controller with CRUD + Disable

```csharp
[ApiController]
[Route("api/[controller]")]
public class ItemsController : ControllerBase
{
    private readonly AppDbContext _context;

    public ItemsController(AppDbContext context)
    {
        _context = context;
    }

    // CREATE
    [HttpPost]
    public async Task<IActionResult> Create(Item item)
    {
        _context.Items.Add(item);
        await _context.SaveChangesAsync();
        return CreatedAtAction(nameof(GetById), new { id = item.Id }, item);
    }

    // READ (Single)
    [HttpGet("{id}")]
    public async Task<IActionResult> GetById(int id)
    {
        var item = await _context.Items.FindAsync(id);
        if (item == null) return NotFound();
        return Ok(item);
    }

    // READ (All active items)
    [HttpGet]
    public async Task<IActionResult> GetAllActive()
    {
        var items = await _context.Items.Where(i => i.IsActive).ToListAsync();
        return Ok(items);
    }

    // UPDATE
    [HttpPut("{id}")]
    public async Task<IActionResult> Update(int id, Item updatedItem)
    {
        if (id != updatedItem.Id)
            return BadRequest();

        var item = await _context.Items.FindAsync(id);
        if (item == null) return NotFound();

        item.Name = updatedItem.Name;
        // Keep IsActive unchanged here or update if needed

        await _context.SaveChangesAsync();
        return NoContent();
    }

    // DELETE (Hard delete)
    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(int id)
    {
        var item = await _context.Items.FindAsync(id);
        if (item == null) return NotFound();

        _context.Items.Remove(item);
        await _context.SaveChangesAsync();
        return NoContent();
    }

    // DISABLE (Soft disable)
    [HttpPatch("{id}/disable")]
    public async Task<IActionResult> Disable(int id)
    {
        var item = await _context.Items.FindAsync(id);
        if (item == null) return NotFound();

        if (!item.IsActive)
            return BadRequest("Item is already disabled.");

        item.IsActive = false;
        await _context.SaveChangesAsync();
        return NoContent();
    }

    // ENABLE (Re-enable disabled item)
    [HttpPatch("{id}/enable")]
    public async Task<IActionResult> Enable(int id)
    {
        var item = await _context.Items.FindAsync(id);
        if (item == null) return NotFound();

        if (item.IsActive)
            return BadRequest("Item is already enabled.");

        item.IsActive = true;
        await _context.SaveChangesAsync();
        return NoContent();
    }
}
```

---

### Explanation

* `IsActive` flag controls whether the item is enabled or disabled.
* `Disable` and `Enable` endpoints update this flag.
* Normal GET only returns enabled items (`GetAllActive`).
* `Delete` is a hard delete that removes the record physically.
* Update doesn’t affect the `IsActive` flag by default, but you can modify it if you want.

---

# 👉Bulk Operation
**bulk CRUD operations** (bulk create, bulk update, bulk disable, and bulk delete) in ASP.NET Core Web API using Entity Framework Core.

---

### Entity (same as before)

```csharp
public class Item
{
    public int Id { get; set; }
    public string Name { get; set; }
    public bool IsActive { get; set; } = true;
}
```

---

### Controller with Bulk CRUD methods

```csharp
[ApiController]
[Route("api/[controller]")]
public class ItemsController : ControllerBase
{
    private readonly AppDbContext _context;

    public ItemsController(AppDbContext context)
    {
        _context = context;
    }

    // BULK CREATE
    [HttpPost("bulk-create")]
    public async Task<IActionResult> BulkCreate([FromBody] List<Item> items)
    {
        if (items == null || items.Count == 0)
            return BadRequest("No items provided for creation.");

        _context.Items.AddRange(items);
        await _context.SaveChangesAsync();
        return Ok(new { CreatedCount = items.Count });
    }

    // BULK UPDATE
    [HttpPut("bulk-update")]
    public async Task<IActionResult> BulkUpdate([FromBody] List<Item> items)
    {
        if (items == null || items.Count == 0)
            return BadRequest("No items provided for update.");

        foreach (var updatedItem in items)
        {
            var existingItem = await _context.Items.FindAsync(updatedItem.Id);
            if (existingItem == null)
                continue; // Or collect errors and return later

            existingItem.Name = updatedItem.Name;
            existingItem.IsActive = updatedItem.IsActive;
            // Update other fields as needed
        }

        await _context.SaveChangesAsync();
        return Ok(new { UpdatedCount = items.Count });
    }

    // BULK DISABLE (soft delete)
    [HttpPatch("bulk-disable")]
    public async Task<IActionResult> BulkDisable([FromBody] List<int> itemIds)
    {
        if (itemIds == null || itemIds.Count == 0)
            return BadRequest("No item IDs provided for disable.");

        var itemsToDisable = await _context.Items.Where(i => itemIds.Contains(i.Id) && i.IsActive).ToListAsync();

        if (itemsToDisable.Count == 0)
            return NotFound("No active items found to disable.");

        itemsToDisable.ForEach(i => i.IsActive = false);

        await _context.SaveChangesAsync();
        return Ok(new { DisabledCount = itemsToDisable.Count });
    }

    // BULK DELETE (hard delete)
    [HttpDelete("bulk-delete")]
    public async Task<IActionResult> BulkDelete([FromBody] List<int> itemIds)
    {
        if (itemIds == null || itemIds.Count == 0)
            return BadRequest("No item IDs provided for deletion.");

        var itemsToDelete = await _context.Items.Where(i => itemIds.Contains(i.Id)).ToListAsync();

        if (itemsToDelete.Count == 0)
            return NotFound("No items found to delete.");

        _context.Items.RemoveRange(itemsToDelete);
        await _context.SaveChangesAsync();
        return Ok(new { DeletedCount = itemsToDelete.Count });
    }
}
```

---

### Summary:

* **Bulk Create:** Accepts a list of new items and inserts them.
* **Bulk Update:** Accepts a list of updated items and updates existing ones.
* **Bulk Disable:** Accepts a list of IDs and sets `IsActive = false` for them.
* **Bulk Delete:** Accepts a list of IDs and permanently deletes those items.

---

# 👉implementation of single and multiple image upload CRUD API with your existing Identity + JWT project.

---

# Folder Structure

```
YourProject/
│
├── Controllers/
│   └── ImageController.cs
│
├── Data/
│   └── AppDbContext.cs
│
├── Models/
│   ├── ApplicationUser.cs
│   ├── Image.cs
│   ├── ImageUploadDto.cs
│   └── MultipleImageUploadDto.cs
│
├── wwwroot/
│   └── uploads/          // Folder to store uploaded images
│
├── Program.cs
```

---

# Code files

---

### 1. Models/Image.cs

```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace YourProject.Models
{
    public class Image
    {
        [Key]
        public int Id { get; set; }

        [Required]
        public string FileName { get; set; } = null!;  // Stored file name

        public string OriginalFileName { get; set; } = null!;  // Original name from client

        public DateTime UploadDate { get; set; }

        public string? UserId { get; set; }  // Who uploaded the image
    }
}
```

---

### 2. Models/ImageUploadDto.cs

```csharp
using Microsoft.AspNetCore.Http;
using System.ComponentModel.DataAnnotations;

namespace YourProject.Models
{
    public class ImageUploadDto
    {
        [Required]
        public IFormFile Image { get; set; } = null!;
    }
}
```

---

### 3. Models/MultipleImageUploadDto.cs

```csharp
using Microsoft.AspNetCore.Http;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;

namespace YourProject.Models
{
    public class MultipleImageUploadDto
    {
        [Required]
        public List<IFormFile> Images { get; set; } = new();
    }
}
```

---

### 4. Data/AppDbContext.cs

```csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using YourProject.Models;

namespace YourProject.Data
{
    public class AppDbContext : IdentityDbContext<ApplicationUser>
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

        public DbSet<Image> Images { get; set; }
    }
}
```

---

### 5. Controllers/ImageController.cs

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using System.Security.Claims;
using YourProject.Data;
using YourProject.Models;

namespace YourProject.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    [Authorize]  // Require authentication
    public class ImageController : ControllerBase
    {
        private readonly AppDbContext _context;
        private readonly IWebHostEnvironment _env;

        public ImageController(AppDbContext context, IWebHostEnvironment env)
        {
            _context = context;
            _env = env;
        }

        // Upload one image
        [HttpPost("upload-single")]
        public async Task<IActionResult> UploadSingle([FromForm] ImageUploadDto dto)
        {
            var image = dto.Image;
            if (image == null || image.Length == 0) return BadRequest("No image provided");

            var ext = Path.GetExtension(image.FileName).ToLowerInvariant();
            var allowedExt = new[] { ".jpg", ".jpeg", ".png", ".gif" };
            if (!allowedExt.Contains(ext)) return BadRequest("Unsupported file type");

            var fileName = $"{Guid.NewGuid()}{ext}";
            var path = Path.Combine(_env.WebRootPath, "uploads", fileName);

            await using var stream = new FileStream(path, FileMode.Create);
            await image.CopyToAsync(stream);

            var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);

            var img = new Image
            {
                FileName = fileName,
                OriginalFileName = image.FileName,
                UploadDate = DateTime.UtcNow,
                UserId = userId
            };

            _context.Images.Add(img);
            await _context.SaveChangesAsync();

            return Ok(new { img.Id, img.FileName, img.OriginalFileName });
        }

        // Upload multiple images
        [HttpPost("upload-multiple")]
        public async Task<IActionResult> UploadMultiple([FromForm] MultipleImageUploadDto dto)
        {
            if (dto.Images == null || !dto.Images.Any()) return BadRequest("No images provided");

            var allowedExt = new[] { ".jpg", ".jpeg", ".png", ".gif" };
            var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);

            var result = new List<object>();

            foreach (var image in dto.Images)
            {
                var ext = Path.GetExtension(image.FileName).ToLowerInvariant();
                if (!allowedExt.Contains(ext)) return BadRequest($"Unsupported file: {image.FileName}");

                var fileName = $"{Guid.NewGuid()}{ext}";
                var path = Path.Combine(_env.WebRootPath, "uploads", fileName);

                await using var stream = new FileStream(path, FileMode.Create);
                await image.CopyToAsync(stream);

                var img = new Image
                {
                    FileName = fileName,
                    OriginalFileName = image.FileName,
                    UploadDate = DateTime.UtcNow,
                    UserId = userId
                };

                _context.Images.Add(img);
                await _context.SaveChangesAsync();

                result.Add(new { img.Id, img.FileName, img.OriginalFileName });
            }

            return Ok(result);
        }

        // List all images uploaded by current user
        [HttpGet("list")]
        public async Task<IActionResult> List()
        {
            var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);

            var images = await _context.Images
                .Where(i => i.UserId == userId)
                .Select(i => new
                {
                    i.Id,
                    i.FileName,
                    i.OriginalFileName,
                    i.UploadDate,
                    Url = $"{Request.Scheme}://{Request.Host}/uploads/{i.FileName}"
                })
                .ToListAsync();

            return Ok(images);
        }

        // Delete image by id
        [HttpDelete("delete/{id}")]
        public async Task<IActionResult> Delete(int id)
        {
            var userId = User.FindFirstValue(ClaimTypes.NameIdentifier);
            var image = await _context.Images.FindAsync(id);

            if (image == null || image.UserId != userId) return NotFound("Not found or access denied");

            var path = Path.Combine(_env.WebRootPath, "uploads", image.FileName);

            if (System.IO.File.Exists(path)) System.IO.File.Delete(path);

            _context.Images.Remove(image);
            await _context.SaveChangesAsync();

            return Ok("Deleted successfully");
        }
    }
}
```

---

### 6. Program.cs (partial)

```csharp
app.UseStaticFiles();  // serve wwwroot files including /uploads

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
```

---

# Summary

* **Single upload:** POST `/api/image/upload-single` with form-data `image`
* **Multiple upload:** POST `/api/image/upload-multiple` with form-data `images` (multiple)
* **List images:** GET `/api/image/list` (returns URLs)
* **Delete image:** DELETE `/api/image/delete/{id}`

---

<br>
<br>
<br>
<br>

# 👉 Search and filter

---

# Folder structure (minimal)

```
YourProject/
├── Controllers/
│   └── ProductController.cs
├── Models/
│   ├── Product.cs
│   └── ProductFilterDto.cs
├── Data/
│   └── AppDbContext.cs
```

---

# 1. Models/Product.cs

```csharp
using System.ComponentModel.DataAnnotations;

namespace YourProject.Models
{
    public class Product
    {
        [Key]
        public int Id { get; set; }

        [Required]
        public string Name { get; set; } = null!;

        public string? Description { get; set; }

        public decimal Price { get; set; }

        public string? Category { get; set; }

        public string? Brand { get; set; }

        public DateTime CreatedAt { get; set; }
    }
}
```

---

# 2. Models/ProductFilterDto.cs

```csharp
namespace YourProject.Models
{
    public class ProductFilterDto
    {
        public string? SearchTerm { get; set; }    // search by name or description
        public string? Category { get; set; }      // filter by category
        public string? Brand { get; set; }         // filter by brand
        public decimal? MinPrice { get; set; }     // price range filter
        public decimal? MaxPrice { get; set; }
    }
}
```

---

# 3. Data/AppDbContext.cs

```csharp
using Microsoft.EntityFrameworkCore;
using YourProject.Models;

namespace YourProject.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

        public DbSet<Product> Products { get; set; }
    }
}
```

---

# 4. Controllers/ProductController.cs

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using YourProject.Data;
using YourProject.Models;

[ApiController]
[Route("api/[controller]")]
public class ProductController : ControllerBase
{
    private readonly AppDbContext _context;

    public ProductController(AppDbContext context)
    {
        _context = context;
    }

    // GET api/product/search-filter
    // Search products by term and filter by category, brand, price range
    [HttpGet("search-filter")]
    public async Task<IActionResult> SearchAndFilter([FromQuery] ProductFilterDto filter)
    {
        var query = _context.Products.AsQueryable();

        // Search: name or description contains search term
        if (!string.IsNullOrWhiteSpace(filter.SearchTerm))
        {
            var term = filter.SearchTerm.ToLower();
            query = query.Where(p => p.Name.ToLower().Contains(term) || (p.Description != null && p.Description.ToLower().Contains(term)));
        }

        // Filter by category
        if (!string.IsNullOrWhiteSpace(filter.Category))
        {
            query = query.Where(p => p.Category == filter.Category);
        }

        // Filter by brand
        if (!string.IsNullOrWhiteSpace(filter.Brand))
        {
            query = query.Where(p => p.Brand == filter.Brand);
        }

        // Filter by price range
        if (filter.MinPrice.HasValue)
        {
            query = query.Where(p => p.Price >= filter.MinPrice.Value);
        }
        if (filter.MaxPrice.HasValue)
        {
            query = query.Where(p => p.Price <= filter.MaxPrice.Value);
        }

        var products = await query.ToListAsync();

        return Ok(products);
    }
}
```

---

# How to test

Request examples:

* Search by keyword:
  `GET /api/product/search-filter?searchTerm=phone`

* Filter by category and brand:
  `GET /api/product/search-filter?category=Electronics&brand=Apple`

* Filter by price range:
  `GET /api/product/search-filter?minPrice=100&maxPrice=1000`

* Combine all:
  `GET /api/product/search-filter?searchTerm=phone&category=Electronics&brand=Apple&minPrice=200&maxPrice=1500`

---

# Summary

* Use `IQueryable` to build the query step-by-step.
* Apply search and filters based on non-null query parameters.
* Return filtered results.

---
