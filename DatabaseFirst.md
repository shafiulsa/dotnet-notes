

---

# **Guide: Scaffolding, DbContext, and Area Integration in ASP.NET Core MVC**

---

## **Case 1: Create a new table in DB and scaffold it completely**

**Scenario:** You have just created a new table `Banner` in your database. You want to generate **model and DbContext** automatically.

### **Folder Structure Before**

```
OnlineShop/
├── Models/
│   └── Db/      <-- empty or does not contain OnlineShopContext yet
```

### **Terminal Command**

```bash
dotnet ef dbcontext scaffold "Server=localhost;Database=OnlineShop;User Id=sa;Password=Your-Password;TrustServerCertificate=True;" Microsoft.EntityFrameworkCore.SqlServer --output-dir Models/Db --context OnlineShopContext --data-annotations --force
```

**Explanation of each part:**

| Option                                                | Meaning                                                        |
| ----------------------------------------------------- | -------------------------------------------------------------- |
| `dotnet ef dbcontext scaffold`                        | Command to generate DbContext + model classes from existing DB |
| `"Server=...;Database=...;User Id=...;Password=...;"` | Your SQL Server connection string                              |
| `Microsoft.EntityFrameworkCore.SqlServer`             | Provider for SQL Server                                        |
| `--output-dir Models/Db`                              | Where to put the generated models                              |
| `--context OnlineShopContext`                         | Name of the DbContext class to create                          |
| `--data-annotations`                                  | Use attributes like `[Key]`, `[StringLength]` in models        |
| `--force`                                             | Overwrite existing files if they exist                         |

### **Folder Structure After**

```
OnlineShop/
├── Models/
│   └── Db/
│       ├── OnlineShopContext.cs
│       └── Banner.cs
```

 You now have a **fully scaffolded DbContext and Banner model**.

---

## **Case 2: Already have a DbContext, add a new table**

**Scenario:** You already have `Menu` table scaffolded and `OnlineShopContext`. You just created a new `Banner` table in the DB. You want **only the model** and then update the existing DbContext manually.

### **Folder Structure Before**

```
OnlineShop/
├── Models/
│   └── Db/
│       ├── OnlineShopContext.cs
│       └── Menu.cs
```

### **Terminal Command to Scaffold Only Banner**

```bash
dotnet ef dbcontext scaffold "Server=localhost;Database=OnlineShop;User Id=sa;Password=Your-Password;TrustServerCertificate=True;" Microsoft.EntityFrameworkCore.SqlServer --table Banner --output-dir Models/Db --data-annotations --no-onconfiguring --force
```

**Explanation of new flags:**

| Option                | Meaning                                             |
| --------------------- | --------------------------------------------------- |
| `--table Banner`      | Scaffold **only this table**, not the whole DB      |
| `--no-onconfiguring`  | Do **not overwrite** connection string in DbContext |
| Others same as Case 1 | Same as before                                      |

### **Update `OnlineShopContext.cs` manually**

```csharp
public virtual DbSet<Banner> Banners { get; set; }

modelBuilder.Entity<Banner>(entity =>
{
    entity.Property(e => e.Title).HasMaxLength(200);
    entity.Property(e => e.SubTitle).HasMaxLength(1000);
    entity.Property(e => e.ImageName).HasMaxLength(50);
    entity.Property(e => e.Link).HasMaxLength(100);
    entity.Property(e => e.Position).HasMaxLength(50);
});
```

### **Folder Structure After**

```
OnlineShop/
├── Models/
│   └── Db/
│       ├── OnlineShopContext.cs  <-- DbSet<Banner> added
│       ├── Menu.cs
│       └── Banner.cs             <-- scaffolded model
```

---

## **Case 3: Use the model and DbContext in an Area (`Admin`)**

**Scenario:** You want to **use the new Banner model inside `Areas/Admin`** for controllers and views.

Great! Since you already have:

 **Model** (e.g., `Banner`)
 **DbContext** (e.g., `OnlineShopContext`)

Now you want to **scaffold the controller and views inside the Admin area**. Here’s how to do it step by step:

---

###  **Folder Structure Before Scaffolding**

```
Areas/
 └── Admin/
     ├── Controllers/
     │    └── HomeController.cs
     ├── Views/
     │    ├── Home/
     │    │    └── Index.cshtml
     │    └── Shared/
     │         ├── _Layout.cshtml
     │         └── _ViewStart.cshtml
Models/
 └── Db/
     ├── OnlineShopContext.cs
     └── Banner.cs
```

---

### **Command to Scaffold Controller & Views**

Run this in **project root**:

```bash
dotnet aspnet-codegenerator controller \
  -name BannersController \
  -m Online_shop.Models.Db.Banner \
  -dc Online_shop.Models.Db.OnlineShopContext \
  --relativeFolderPath Areas/Admin/Controllers \
  --useDefaultLayout \
  --referenceScriptLibraries
```

#### **Explanation of options:**

* `-name BannersController` → Name of the controller.
* `-m Online_shop.Models.Db.Banner` → Fully qualified name of your model.
* `-dc Online_shop.Models.Db.OnlineShopContext` → Fully qualified name of your DbContext.
* `--relativeFolderPath Areas/Admin/Controllers` → Places controller inside `Areas/Admin/Controllers`.
* `--useDefaultLayout` → Uses the site’s default layout.
* `--referenceScriptLibraries` → Adds script references for validation.

---

###  **Move Views to Area**

By default, views will be created under `Views/Banners`. Move them to:

```
Areas/Admin/Views/Banners/
```

---

###  **Modify Controller for Area**

Open **`BannersController.cs`** and add:

```csharp
[Area("Admin")]
public class BannersController : Controller
{
    // existing code
}
```

Also, make sure the **namespace** matches your area, e.g.:

```csharp
namespace Online_shop.Areas.Admin.Controllers
```

---

###  **Update Admin Area Route**

Check `Program.cs` or `Startup.cs` (depending on .NET version) and confirm:

```csharp
app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");
```

---

###  **Folder Structure After Scaffolding**

```
Areas/
 └── Admin/
     ├── Controllers/
     │    ├── HomeController.cs
     │    └── BannersController.cs
     ├── Views/
     │    ├── Home/
     │    │    └── Index.cshtml
     │    ├── Banners/
     │    │    ├── Create.cshtml
     │    │    ├── Delete.cshtml
     │    │    ├── Details.cshtml
     │    │    ├── Edit.cshtml
     │    │    └── Index.cshtml
     │    └── Shared/
     │         ├── _Layout.cshtml
     │         └── _ViewStart.cshtml
Models/
 └── Db/
     ├── OnlineShopContext.cs
     └── Banner.cs
```

---

 Now your `Admin/Banners` section will be accessible at:

```
http://localhost:5041/Admin/Banners
```

---
## **More clearly the concept **




Want the **CommentsController** scaffolding for the Admin area in the same format as your Banner example. Here’s how it will look:

---

### **Folder Structure Before Scaffolding**

```
Areas/
 └── Admin/
     ├── Controllers/
     │    └── HomeController.cs
     ├── Views/
     │    ├── Home/
     │    │    └── Index.cshtml
     │    └── Shared/
     │         ├── _Layout.cshtml
     │         └── _ViewStart.cshtml
Models/
 └── Db/
     ├── OnlineShopContext.cs
     └── Comment.cs
```

---

### **Command to Scaffold Controller & Views**

Run this in the **project root**:

```bash
dotnet aspnet-codegenerator controller \
  -name CommentsController \
  -m OnlineShop.Models.Db.Comment \
  -dc OnlineShop.Models.Db.OnlineShopContext \
  --relativeFolderPath Areas/Admin/Controllers \
  --layout ~/Areas/Admin/Views/Shared/_Layout.cshtml \
  --referenceScriptLibraries \
  -f
```

 **Explanation of options:**

* `-name CommentsController` → Name of the controller.
* `-m OnlineShop.Models.Db.Comment` → Fully qualified name of your model.
* `-dc OnlineShop.Models.Db.OnlineShopContext` → Fully qualified name of your DbContext.
* `--relativeFolderPath Areas/Admin/Controllers` → Places controller inside Admin area.
* `--layout ~/Areas/Admin/Views/Shared/_Layout.cshtml` → Uses Admin area layout.
* `--referenceScriptLibraries` → Adds validation JS libraries in the views.
* `-f` → Force overwrite if files already exist.

---

### **Modify Controller for Area**

Open `CommentsController.cs` and add:

```csharp
[Area("Admin")]
public class CommentsController : Controller
{
    // scaffolded code will be here
}
```

Also, update the namespace:

```csharp
namespace OnlineShop.Areas.Admin.Controllers
```

---

### **Update Admin Area Route**

Check **Program.cs** or **Startup.cs**:

```csharp
app.MapControllerRoute(
    name: "areas",
    pattern: "{area:exists}/{controller=Home}/{action=Index}/{id?}");
```

---

### **Folder Structure After Scaffolding**

```
Areas/
 └── Admin/
     ├── Controllers/
     │    ├── HomeController.cs
     │    └── CommentsController.cs
     ├── Views/
     │    ├── Home/
     │    │    └── Index.cshtml
     │    ├── Comments/
     │    │    ├── Create.cshtml
     │    │    ├── Delete.cshtml
     │    │    ├── Details.cshtml
     │    │    ├── Edit.cshtml
     │    │    └── Index.cshtml
     │    └── Shared/
     │         ├── _Layout.cshtml
     │         └── _ViewStart.cshtml
Models/
 └── Db/
     ├── OnlineShopContext.cs
     └── Comment.cs
```

---

### **Access**

After scaffolding, your **Admin Comments section** will be accessible at:

```
http://localhost:5041/Admin/Comments
```






---

##  **Key Notes About the Scaffold Command You Mentioned**

```bash
dotnet ef dbcontext scaffold "Server=localhost;Database=OnlineShop;User Id=sa;Password=Your-Password;TrustServerCertificate=True;" Microsoft.EntityFrameworkCore.SqlServer --table Banner --output-dir Models/Db --data-annotations --no-onconfiguring --force
```

* **Purpose:** Scaffold a **single table** into a model class, **without overwriting the existing DbContext**
* **Use Case:** Adding a new table to an existing project without touching other models or the existing context
* **Important Flags:**

  * `--table Banner` → only this table
  * `--output-dir Models/Db` → put model under Models/Db
  * `--no-onconfiguring` → prevent changing existing DbContext connection
  * `--force` → overwrite if file exists

---


