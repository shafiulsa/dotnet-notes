

# Class Diagram Overview

```plaintext
+----------------+        1        1 +-----------------+
|     User       |--------------------|  UserProfile    |
+----------------+                    +-----------------+
| - Id           |                    | - Id            |
| - Email        |                    | - FullName      |
|                |                    | - Address       |
| +Profile       |                    | - UserId (FK)   |
| +Orders        | 1              *   +-----------------+
+----------------+---------------------+
        |
        | 1
        |                             
        | *                               
+----------------+        *       1 +------------------+
|    Order       |------------------|      User        |
+----------------+                  +------------------+
| - Id           |                  | (already shown)  |
| - OrderDate    |
| - Status       |                                      
| - UserId (FK)  |                                      
| +Items         |                                      
+----------------+                                      
        | 1                                                 
        |                                                    
        | *                                                  
+----------------+        *       1 +-----------------+    
|  OrderItem     |-------------------|   Product       |    
+----------------+                   +-----------------+    
| - Id           |                   | - Id            |    
| - Quantity     |                   | - Title         |    
| - UnitPrice    |                   | - Price         |    
| - ProductId(FK)|                   | - CategoryId(FK)|
| - OrderId (FK) |                   | +Category       |    
+----------------+                   | +ProductTags    |    
                                   +-----------------+    
                                            | 1
                                            |
                                            | *
+----------------+         *          *     +-----------------+
|    Category    |-------------------------|     ProductTag   |
+----------------+                         +-----------------+
| - Id           |                         | - ProductId (FK) |
| - Name         |                         | - TagId (FK)     |
| +Products      |                         | - AssignedAt     |
+----------------+                         +-----------------+
                                                  | *
                                                  |
                                                  | 1
                                        +-----------------+
                                        |      Tag        |
                                        +-----------------+
                                        | - Id            |
                                        | - Name          |
                                        | +ProductTags    |
                                        +-----------------+
```

---

# Explanation:

* **User** has a **one-to-one** relationship with **UserProfile** (each user has one profile).
* **User** has **one-to-many** relationship with **Order** (a user can place multiple orders).
* **Order** has **one-to-many** relationship with **OrderItem** (each order contains multiple order items).
* **OrderItem** has **many-to-one** relationship with **Product** (each order item references one product).
* **Product** has **many-to-one** relationship with **Category** (each product belongs to one category).
* **Product** and **Tag** have a **many-to-many** relationship via the join entity **ProductTag**.

---

Got it! You want a **moderate-level e-commerce project structure and code** **without Clean Architecture**, i.e., a typical layered structure often used in many ASP.NET Core projects that is simpler but practical.

---

# Masterpiece Note: Moderate-Level E-Commerce Project (Non-Clean Architecture)

---

## 1️⃣ Relationship Intuition & Examples

| Relationship Type | Intuition                                                   | E-commerce Example                               |
| ----------------- | ----------------------------------------------------------- | ------------------------------------------------ |
| **One-to-One**    | One entity instance maps to exactly one instance of another | `User` ↔ `UserProfile` (each user has a profile) |
| **One-to-Many**   | One parent entity relates to many child entities            | `Category` → multiple `Products`                 |
| **Many-to-One**   | Many child entities relate to one parent                    | `Product` belongs to one `Category`              |
| **Many-to-Many**  | Entities relate to many others via a join table             | `Product` ↔ `Tag`                                |

---

## 2️⃣ Folder Structure (Non-Clean layered typical ASP.NET Core MVC / Web API)

```
/ECommerceProject
 ├── /Controllers
 │     ├── UserController.cs
 │     ├── ProductController.cs
 │     ├── CategoryController.cs
 │     └── OrderController.cs
 ├── /Models
 │     ├── User.cs
 │     ├── UserProfile.cs
 │     ├── Category.cs
 │     ├── Product.cs
 │     ├── Tag.cs
 │     ├── ProductTag.cs
 │     ├── Order.cs
 │     ├── OrderItem.cs
 │     └── Enums.cs
 ├── /Data
 │     └── ApplicationDbContext.cs
 ├── /DTOs
 │     ├── CreateUserDto.cs
 │     ├── CreateProductDto.cs
 │     └── CreateOrderDto.cs
 ├── /Services
 │     └── (Optional service classes if you want separation of business logic)
 ├── Program.cs
 ├── Startup.cs (if using ASP.NET Core 3.x / 5)
 └── appsettings.json
```

---

## 3️⃣ Models with Comments (inside `/Models`)

---

### `/Models/User.cs`

```csharp
using System.Collections.Generic;

namespace ECommerceProject.Models
{
    // User entity - stores login info & relations
    public class User
    {
        public int Id { get; set; }
        public string Email { get; set; }

        // One-to-One relationship with UserProfile
        public UserProfile Profile { get; set; }

        // One-to-Many: User can have multiple Orders
        public ICollection<Order> Orders { get; set; }
    }
}
```

---

### `/Models/UserProfile.cs`

```csharp
namespace ECommerceProject.Models
{
    // Profile with additional user info
    public class UserProfile
    {
        public int Id { get; set; }
        public string FullName { get; set; }
        public string Address { get; set; }

        // FK to User
        public int UserId { get; set; }
        public User User { get; set; }
    }
}
```

---

### `/Models/Category.cs`

```csharp
using System.Collections.Generic;

namespace ECommerceProject.Models
{
    // Product category
    public class Category
    {
        public int Id { get; set; }
        public string Name { get; set; }

        // One-to-many: Category has many Products
        public ICollection<Product> Products { get; set; }
    }
}
```

---

### `/Models/Product.cs`

```csharp
using System.Collections.Generic;

namespace ECommerceProject.Models
{
    // Product entity with Category and Tags
    public class Product
    {
        public int Id { get; set; }
        public string Title { get; set; }
        public decimal Price { get; set; }

        // Many-to-one relationship: belongs to Category
        public int CategoryId { get; set; }
        public Category Category { get; set; }

        // Many-to-many with Tags via join entity ProductTag
        public ICollection<ProductTag> ProductTags { get; set; }
    }
}
```

---

### `/Models/Tag.cs`

```csharp
using System.Collections.Generic;

namespace ECommerceProject.Models
{
    // Tag entity for categorization
    public class Tag
    {
        public int Id { get; set; }
        public string Name { get; set; }

        public ICollection<ProductTag> ProductTags { get; set; }
    }
}
```

---

### `/Models/ProductTag.cs`

```csharp
using System;

namespace ECommerceProject.Models
{
    // Join table for Product <-> Tag many-to-many
    public class ProductTag
    {
        public int ProductId { get; set; }
        public Product Product { get; set; }

        public int TagId { get; set; }
        public Tag Tag { get; set; }

        public DateTime AssignedAt { get; set; } = DateTime.UtcNow;
    }
}
```

---

### `/Models/Enums.cs`

```csharp
namespace ECommerceProject.Models
{
    // Enum for order status
    public enum OrderStatus
    {
        Pending,
        Processing,
        Shipped,
        Delivered,
        Cancelled
    }
}
```

---

### `/Models/Order.cs`

```csharp
using System;
using System.Collections.Generic;

namespace ECommerceProject.Models
{
    public class Order
    {
        public int Id { get; set; }
        public DateTime OrderDate { get; set; }

        public OrderStatus Status { get; set; }

        // Many orders belong to one User
        public int UserId { get; set; }
        public User User { get; set; }

        // One order has many order items
        public ICollection<OrderItem> Items { get; set; }
    }
}
```

---

### `/Models/OrderItem.cs`

```csharp
namespace ECommerceProject.Models
{
    public class OrderItem
    {
        public int Id { get; set; }
        public int Quantity { get; set; }
        public decimal UnitPrice { get; set; }

        // Many order items belong to one Product
        public int ProductId { get; set; }
        public Product Product { get; set; }

        // Many order items belong to one Order
        public int OrderId { get; set; }
        public Order Order { get; set; }
    }
}
```

---

## 4️⃣ DbContext (in `/Data/ApplicationDbContext.cs`)

```csharp
using Microsoft.EntityFrameworkCore;
using ECommerceProject.Models;

namespace ECommerceProject.Data
{
    public class ApplicationDbContext : DbContext
    {
        public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
            : base(options) { }

        public DbSet<User> Users { get; set; }
        public DbSet<UserProfile> UserProfiles { get; set; }
        public DbSet<Category> Categories { get; set; }
        public DbSet<Product> Products { get; set; }
        public DbSet<Tag> Tags { get; set; }
        public DbSet<ProductTag> ProductTags { get; set; }
        public DbSet<Order> Orders { get; set; }
        public DbSet<OrderItem> OrderItems { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            // One-to-one User -> UserProfile
            modelBuilder.Entity<User>()
                .HasOne(u => u.Profile)
                .WithOne(p => p.User)
                .HasForeignKey<UserProfile>(p => p.UserId);

            // One-to-many Category -> Products
            modelBuilder.Entity<Category>()
                .HasMany(c => c.Products)
                .WithOne(p => p.Category)
                .HasForeignKey(p => p.CategoryId);

            // Many-to-many Product <-> Tag using join table ProductTag
            modelBuilder.Entity<ProductTag>()
                .HasKey(pt => new { pt.ProductId, pt.TagId });

            modelBuilder.Entity<ProductTag>()
                .HasOne(pt => pt.Product)
                .WithMany(p => p.ProductTags)
                .HasForeignKey(pt => pt.ProductId);

            modelBuilder.Entity<ProductTag>()
                .HasOne(pt => pt.Tag)
                .WithMany(t => t.ProductTags)
                .HasForeignKey(pt => pt.TagId);

            // One-to-many Order -> OrderItems
            modelBuilder.Entity<Order>()
                .HasMany(o => o.Items)
                .WithOne(oi => oi.Order)
                .HasForeignKey(oi => oi.OrderId);

            // Store enum as string in DB for readability
            modelBuilder.Entity<Order>()
                .Property(o => o.Status)
                .HasConversion<string>();
        }
    }
}
```

---

## 5️⃣ DTOs (in `/DTOs` folder) — Example for Product

### `/DTOs/CreateProductDto.cs`

```csharp
using System.Collections.Generic;

namespace ECommerceProject.DTOs
{
    public class CreateProductDto
    {
        public string Title { get; set; }
        public decimal Price { get; set; }
        public int CategoryId { get; set; }

        // List of tag IDs for many-to-many relation
        public List<int> TagIds { get; set; }
    }
}
```

---

## 6️⃣ Example Controller for Products (`/Controllers/ProductController.cs`)

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using ECommerceProject.Data;
using ECommerceProject.DTOs;
using ECommerceProject.Models;
using System.Linq;
using System.Threading.Tasks;

namespace ECommerceProject.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ProductController : ControllerBase
    {
        private readonly ApplicationDbContext _db;

        public ProductController(ApplicationDbContext db)
        {
            _db = db;
        }

        // POST: api/product
        [HttpPost]
        public async Task<IActionResult> CreateProduct([FromBody] CreateProductDto dto)
        {
            // Validate category exists
            var categoryExists = await _db.Categories.AnyAsync(c => c.Id == dto.CategoryId);
            if (!categoryExists)
                return BadRequest("Category does not exist");

            // Validate tags exist
            var validTagIds = await _db.Tags
                .Where(t => dto.TagIds.Contains(t.Id))
                .Select(t => t.Id)
                .ToListAsync();

            if (validTagIds.Count != dto.TagIds.Count)
                return BadRequest("One or more tags are invalid");

            var product = new Product
            {
                Title = dto.Title,
                Price = dto.Price,
                CategoryId = dto.CategoryId,
                ProductTags = validTagIds.Select(id => new ProductTag { TagId = id }).ToList()
            };

            _db.Products.Add(product);
            await _db.SaveChangesAsync();

            return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
        }

        // GET: api/product/{id}
        [HttpGet("{id}")]
        public async Task<IActionResult> GetProduct(int id)
        {
            var product = await _db.Products
                .Include(p => p.Category)
                .Include(p => p.ProductTags)
                    .ThenInclude(pt => pt.Tag)
                .FirstOrDefaultAsync(p => p.Id == id);

            if (product == null)
                return NotFound();

            return Ok(product);
        }
    }
}
```

---

## 7️⃣ Summary of Key Concepts

| Topic                         | Explanation                                                                            |
| ----------------------------- | -------------------------------------------------------------------------------------- |
| **One-to-One**                | Separate tightly coupled data into different tables with FK in dependent entity        |
| **One-to-Many / Many-to-One** | Parent-child relationship via FK in child table                                        |
| **Many-to-Many**              | Use explicit join table to relate many-to-many entities with possibility of extra info |
| **DTOs**                      | Data Transfer Objects control what API accepts/returns, prevent over-posting           |
| **Navigation Properties**     | Define entity relations to allow EF Core to load related data easily with `.Include()` |
| **Fluent API Config**         | Explicitly configure relationships, keys, enum conversion for database clarity         |
| **Folder Structure**          | Separate concerns: Models, Data (DbContext), Controllers, DTOs for maintainability     |

---



