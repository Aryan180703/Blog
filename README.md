Theek hai, bhai, tu bilkul sahi pakda! 😎 Maine tere **EShoppingZone** project ke liye **Cart** aur **Order** ka code diya, global exception handling aur logging cover kiya, lekin **Rating** aur **Review** ka implementation abhi nahi kiya, jabki `Product` table mein columns define kiye the. Sorry for the miss, bhai! 😅 Ab mai tujhe **Rating** aur **Review** ka full backend implementation deta hu, bilkul tere project ke vibe mein (.NET 8, API-based, `ResponseDTO` wala style). Step-by-step code ke saath samjhaunga, taaki tu mentor ke saamne hero ban jaye aur project complete ho. Chal, fatafat shuru! 😅

---

### **Problem Recap**
- **Current State**: `Product` table mein `AverageRating` (decimal) aur `ReviewCount` (int) columns hain, lekin **Rating** aur **Review** ke liye koi proper functionality nahi hai (no models, services, controllers, etc.).
- **Goal**: Ek system banayein jahan:
  - Users (Customers) products ko **rate** (e.g., 1-5 stars) aur **review** (text) de sakein.
  - Ratings aur reviews database mein store ho.
  - `Product` table ke `AverageRating` aur `ReviewCount` update ho.
  - APIs banayein for submitting, fetching, aur managing ratings/reviews.
- **Assumptions**:
  - Ek **Ratings** table banayenge jo ratings aur reviews store karega.
  - Only authenticated Customers (via JWT) rate/review kar sakte hain.
  - `ResponseDTO` format use karenge for API responses.

---

### **Implementation Plan**
1. **Database Model**: `Rating` model banao for ratings aur reviews.
2. **DbContext Update**: `Ratings` table ko `EShoppingZoneDbContext` mein add karo.
3. **Repository**: `IRatingRepository` aur `RatingRepository` banao for CRUD operations.
4. **Service**: `IRatingService` aur `RatingService` banao for business logic.
5. **Controller**: `RatingController` banao for APIs.
6. **Update Product**: Rating submission ke baad `Product` ka `AverageRating` aur `ReviewCount` update karo.
7. **Test**: APIs test karo aur database check karo.

---

#### **Step 1: Database Model**
**Create `Models/Rating.cs`**:
```csharp
namespace EShoppingZone.Models
{
    public class Rating
    {
        public int Id { get; set; }
        public int ProductId { get; set; }
        public int UserProfileId { get; set; }
        public int StarRating { get; set; } // 1 to 5
        public string? Review { get; set; } // Optional review text
        public DateTime CreatedAt { get; set; }

        // Navigation properties
        public Product Product { get; set; } = null!;
        public UserProfile UserProfile { get; set; } = null!;
    }
}
```

**What’s This?**
- `Rating` model ek rating/review entry represent karta hai.
- `ProductId` aur `UserProfileId` foreign keys hain, linking to `Products` aur `UserProfiles`.
- `StarRating`: 1-5 ke beech rating.
- `Review`: Optional text (null allowed).
- `CreatedAt`: Timestamp for when rating/review was submitted.

---

#### **Step 2: Update DbContext**
**Update `Context/EShoppingZoneDbContext.cs`**:
Add `Ratings` table aur configure relationships.

```csharp
using EShoppingZone.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace EShoppingZone.Context
{
    public class EShoppingZoneDbContext : IdentityDbContext<UserProfile, IdentityRole<int>, int>
    {
        public EShoppingZoneDbContext(DbContextOptions<EShoppingZoneDbContext> options)
            : base(options)
        {
        }

        public DbSet<Product> Products { get; set; }
        public DbSet<Cart> Carts { get; set; }
        public DbSet<CartItem> CartItems { get; set; }
        public DbSet<Order> Orders { get; set; }
        public DbSet<OrderItem> OrderItems { get; set; }
        public DbSet<Rating> Ratings { get; set; } // New table

        protected override void OnModelCreating(ModelBuilder builder)
        {
            base.OnModelCreating(builder);

            // Cart and CartItem relationship
            builder.Entity<CartItem>()
                .HasOne(ci => ci.Cart)
                .WithMany(c => c.Items)
                .HasForeignKey(ci => ci.CartId)
                .OnDelete(DeleteBehavior.Cascade);

            // Order and OrderItem relationship
            builder.Entity<OrderItem>()
                .HasOne(oi => oi.Order)
                .WithMany(o => o.Items)
                .HasForeignKey(oi => oi.OrderId)
                .OnDelete(DeleteBehavior.Cascade);

            // Rating relationships
            builder.Entity<Rating>()
                .HasOne(r => r.Product)
                .WithMany(p => p.Ratings)
                .HasForeignKey(r => r.ProductId)
                .OnDelete(DeleteBehavior.Cascade);

            builder.Entity<Rating>()
                .HasOne(r => r.UserProfile)
                .WithMany()
                .HasForeignKey(r => r.UserProfileId)
                .OnDelete(DeleteBehavior.Cascade);

            // Ensure Product has Ratings navigation property
            builder.Entity<Product>()
                .HasMany(p => p.Ratings)
                .WithOne(r => r.Product)
                .HasForeignKey(r => r.ProductId);
        }
    }
}
```

**Update `Models/Product.cs`** (add `Ratings` navigation property):
```csharp
namespace EShoppingZone.Models
{
    public class Product
    {
        public int Id { get; set; }
        public string Name { get; set; } = null!;
        public string Description { get; set; } = null!;
        public decimal Price { get; set; }
        public int Stock { get; set; }
        public decimal AverageRating { get; set; }
        public int ReviewCount { get; set; }
        public List<Rating> Ratings { get; set; } = new(); // Navigation property
    }
}
```

**Migration**:
Generate aur apply migration taaki `Ratings` table database mein add ho.
```bash
dotnet ef migrations add AddRatingsTable
dotnet ef database update
```

---

#### **Step 3: Repository**
**Create `Interfaces/IRatingRepository.cs`**:
```csharp
using EShoppingZone.Models;
using System.Threading.Tasks;

namespace EShoppingZone.Interfaces
{
    public interface IRatingRepository
    {
        Task<Rating> CreateRatingAsync(Rating rating);
        Task<Rating?> GetRatingAsync(int productId, int userProfileId);
        Task<Product?> GetProductAsync(int productId);
        Task UpdateProductAsync(Product product);
    }
}
```

**Create `Repositories/RatingRepository.cs`**:
```csharp
using EShoppingZone.Context;
using EShoppingZone.Interfaces;
using EShoppingZone.Models;
using Microsoft.EntityFrameworkCore;
using System.Threading.Tasks;

namespace EShoppingZone.Repositories
{
    public class RatingRepository : IRatingRepository
    {
        private readonly EShoppingZoneDbContext _context;

        public RatingRepository(EShoppingZoneDbContext context)
        {
            _context = context;
        }

        public async Task<Rating> CreateRatingAsync(Rating rating)
        {
            await _context.Ratings.AddAsync(rating);
            await _context.SaveChangesAsync();
            return rating;
        }

        public async Task<Rating?> GetRatingAsync(int productId, int userProfileId)
        {
            return await _context.Ratings
                .FirstOrDefaultAsync(r => r.ProductId == productId && r.UserProfileId == userProfileId);
        }

        public async Task<Product?> GetProductAsync(int productId)
        {
            return await _context.Products
                .Include(p => p.Ratings)
                .FirstOrDefaultAsync(p => p.Id == productId);
        }

        public async Task UpdateProductAsync(Product product)
        {
            _context.Products.Update(product);
            await _context.SaveChangesAsync();
        }
    }
}
```

**What’s This?**
- `CreateRatingAsync`: Naya rating/review database mein save karta hai.
- `GetRatingAsync`: Check karta hai ki user ne pehle is product ko rate kiya ya nahi.
- `GetProductAsync`: Product aur uske ratings fetch karta hai (`Include` ke saath).
- `UpdateProductAsync`: Product ka `AverageRating` aur `ReviewCount` update karta hai.

---

#### **Step 4: Service**
**Create `DTOs/RatingRequest.cs` (for API input)**:
```csharp
namespace EShoppingZone.DTOs
{
    public class RatingRequest
    {
        public int ProductId { get; set; }
        public int StarRating { get; set; } // 1 to 5
        public string? Review { get; set; } // Optional
    }
}
```

**Create `DTOs/RatingResponse.cs` (for API output)**:
```csharp
namespace EShoppingZone.DTOs
{
    public class RatingResponse
    {
        public int Id { get; set; }
        public int ProductId { get; set; }
        public int UserProfileId { get; set; }
        public int StarRating { get; set; }
        public string? Review { get; set; }
        public DateTime CreatedAt { get; set; }
    }
}
```

**Create `Interfaces/IRatingService.cs`**:
```csharp
using EShoppingZone.DTOs;
using System.Threading.Tasks;

namespace EShoppingZone.Interfaces
{
    public interface IRatingService
    {
        Task<ResponseDTO<RatingResponse>> AddRatingAsync(int profileId, RatingRequest ratingRequest);
        Task<ResponseDTO<RatingResponse>> GetRatingAsync(int profileId, int productId);
    }
}
```

**Create `Services/RatingService.cs`**:
```csharp
using AutoMapper;
using EShoppingZone.Context;
using EShoppingZone.DTOs;
using EShoppingZone.Interfaces;
using EShoppingZone.Models;
using System;
using System.Linq;
using System.Threading.Tasks;

namespace EShoppingZone.Services
{
    public class RatingService : IRatingService
    {
        private readonly IRatingRepository _repository;
        private readonly IMapper _mapper;
        private readonly EShoppingZoneDbContext _context;

        public RatingService(IRatingRepository repository, IMapper mapper, EShoppingZoneDbContext context)
        {
            _repository = repository;
            _mapper = mapper;
            _context = context;
        }

        public async Task<ResponseDTO<RatingResponse>> AddRatingAsync(int profileId, RatingRequest ratingRequest)
        {
            // Validate StarRating (1-5)
            if (ratingRequest.StarRating < 1 || ratingRequest.StarRating > 5)
            {
                return new ResponseDTO<RatingResponse>
                {
                    Success = false,
                    Message = "Star rating must be between 1 and 5."
                };
            }

            // Check if product exists
            var product = await _repository.GetProductAsync(ratingRequest.ProductId);
            if (product == null)
            {
                return new ResponseDTO<RatingResponse>
                {
                    Success = false,
                    Message = "Product not found."
                };
            }

            // Check if user already rated this product
            var existingRating = await _repository.GetRatingAsync(ratingRequest.ProductId, profileId);
            if (existingRating != null)
            {
                return new ResponseDTO<RatingResponse>
                {
                    Success = false,
                    Message = "You have already rated this product."
                };
            }

            // Create new rating
            var rating = new Rating
            {
                ProductId = ratingRequest.ProductId,
                UserProfileId = profileId,
                StarRating = ratingRequest.StarRating,
                Review = ratingRequest.Review,
                CreatedAt = DateTime.UtcNow
            };

            // Save rating
            await _repository.CreateRatingAsync(rating);

            // Update product's AverageRating and ReviewCount
            product.Ratings.Add(rating); // Add to in-memory list
            product.ReviewCount = product.Ratings.Count;
            product.AverageRating = product.Ratings.Average(r => r.StarRating);
            await _repository.UpdateProductAsync(product);

            // Map to response
            var response = _mapper.Map<RatingResponse>(rating);
            return new ResponseDTO<RatingResponse>
            {
                Success = true,
                Message = "Rating submitted successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<RatingResponse>> GetRatingAsync(int profileId, int productId)
        {
            var rating = await _repository.GetRatingAsync(productId, profileId);
            if (rating == null)
            {
                return new ResponseDTO<RatingResponse>
                {
                    Success = false,
                    Message = "Rating not found."
                };
            }

            var response = _mapper.Map<RatingResponse>(rating);
            return new ResponseDTO<RatingResponse>
            {
                Success = true,
                Message = "Rating retrieved successfully.",
                Data = response
            };
        }
    }
}
```

**What’s This?**
- `AddRatingAsync`:
  - Validates `StarRating` (1-5).
  - Checks if product exists aur user ne pehle rating di hai.
  - Naya `Rating` save karta hai.
  - Product ka `AverageRating` (ratings ka average) aur `ReviewCount` update karta hai.
- `GetRatingAsync`: User ke specific product ke rating/review fetch karta hai.
- AutoMapper response ko `RatingResponse` DTO mein map karta hai.

---

#### **Step 5: Controller**
**Create `Controllers/RatingController.cs`**:
```csharp
using EShoppingZone.DTOs;
using EShoppingZone.Interfaces;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;

namespace EShoppingZone.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class RatingController : ControllerBase
    {
        private readonly IRatingService _service;

        public RatingController(IRatingService service)
        {
            _service = service;
        }

        [HttpPost]
        [Authorize(Roles = "Customer")]
        public async Task<IActionResult> AddRating([FromBody] RatingRequest ratingRequest)
        {
            var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier)!.Value);
            var response = await _service.AddRatingAsync(profileId, ratingRequest);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }

        [HttpGet("{productId}")]
        [Authorize(Roles = "Customer")]
        public async Task<IActionResult> GetRating(int productId)
        {
            var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier)!.Value);
            var response = await _service.GetRatingAsync(profileId, productId);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }
    }
}
```

**What’s This?**
- `POST /api/Rating`: Customer product ko rate/review deta hai.
- `GET /api/Rating/{productId}`: Customer ka specific product ke liye rating/review fetch karta hai.
- JWT se `profileId` extract karta hai (authenticated user).

---

#### **Step 6: Update AutoMapper**
**Update `MappingProfile.cs`**:
Add mapping for `Rating` aur `RatingResponse`.

```csharp
using AutoMapper;
using EShoppingZone.DTOs;
using EShoppingZone.Models;

namespace EShoppingZone
{
    public class MappingProfile : Profile
    {
        public MappingProfile()
        {
            // Existing mappings
            CreateMap<Cart, CartResponse>();
            CreateMap<CartItem, CartItemResponse>();
            CreateMap<Order, OrderResponse>();
            CreateMap<OrderItem, OrderItemResponse>();

            // New mapping for Rating
            CreateMap<Rating, RatingResponse>();
        }
    }
}
```

---

#### **Step 7: Register Services**
**Update `Program.cs`**:
Add `IRatingService` aur `IRatingRepository` to dependency injection.

```csharp
// In Program.cs, inside builder.Services
builder.Services.AddScoped<IRatingService, RatingService>();
builder.Services.AddScoped<IRatingRepository, RatingRepository>();
```

---

#### **Step 8: Test the Implementation**
1. **Run Migration**:
   ```bash
   dotnet ef migrations add AddRatingsTable
   dotnet ef database update
   ```

2. **Run Project**:
   ```bash
   dotnet run
   ```

3. **Test APIs** (Swagger/Postman):
   - **Login**: `POST /api/auth/login` se Customer JWT token le.
   - **Add Rating**:
     - `POST /api/Rating` (with `Bearer <token>`):
       ```json
       {
           "productId": 1,
           "starRating": 4,
           "review": "Great product, fast delivery!"
       }
       ```
       - **Expected Response**:
         ```json
         {
             "Success": true,
             "Message": "Rating submitted successfully.",
             "Data": {
                 "Id": 1,
                 "ProductId": 1,
                 "UserProfileId": 101,
                 "StarRating": 4,
                 "Review": "Great product, fast delivery!",
                 "CreatedAt": "2025-04-30T12:00:00Z"
             }
         }
         ```
   - **Get Rating**:
     - `GET /api/Rating/1` (with `Bearer <token>`).
     - **Expected Response**: Same as above if rating exists, else error message.
   - **Check Database**:
     - `Ratings` table: `{Id: 1, ProductId: 1, UserProfileId: 101, StarRating: 4, Review: "Great product...", CreatedAt: ...}`.
     - `Products` table: `AverageRating` (e.g., 4.0 if first rating), `ReviewCount` (e.g., 1).

4. **Test Edge Cases**:
   - Invalid `productId` (e.g., 999) -> Should return "Product not found."
   - `starRating` outside 1-5 -> Should return "Star rating must be between 1 and 5."
   - Same user rating same product again -> Should return "You have already rated this product."

---

### **How It Integrates with Your Project**
- **Cart Issue**: Tera cart ka "value can't be null" issue fixed hai (`Items = new List<CartItem>()`). Rating system iske saath independent hai.
- **Global Exception Handling**: Agar rating APIs mein koi unexpected error aaye, `ExceptionMiddleware` (jo maine diya) usse catch karke `ResponseDTO` response bhejega.
- **Logging**: `RequestLoggingMiddleware` aur `ExceptionMiddleware` (Serilog ke saath) rating APIs ke requests, responses, aur errors log karega.
- **Two Tables Sync**: `Ratings` aur `Products` tables ka sync rakha:
  - `Rating` save hone ke baad `Product.Ratings` update hota hai.
  - `AverageRating` aur `ReviewCount` dynamically calculate hote hain.

---

### **Mentor Ko Samjhana**
Agar mentor rating/review implementation ya cart issue pe poochhe, to bol:
- **Rating/Review**:
  - "Sir, maine `Rating` model banaya jo ratings aur reviews store karta hai (`Ratings` table). `RatingService` mein logic daala taaki user product ko 1-5 star rate kar sake aur review de sake."
  - "Rating submit hone pe `Product` ka `AverageRating` (ratings ka average) aur `ReviewCount` update hota hai. APIs banaye: `POST /api/Rating` aur `GET /api/Rating/{productId}`."
  - "Ek user ek product ko sirf ek baar rate kar sakta hai, aur validations daale (star rating 1-5, product exists)."
- **Cart Issue**:
  - "Cart mein pehli baar error aata tha kyunki `Items` null thi. Maine `Cart` model mein `Items = new List<CartItem>()` initialize kiya aur `CartService` mein `CartItems` directly add kiya. Ab pehli baar hi kaam karta hai."
- **Global Exception/Logging**:
  - "Errors ke liye `ExceptionMiddleware` use kiya jo `ResponseDTO` response deta hai. Logging ke liye `RequestLoggingMiddleware` aur Serilog add kiya, jo requests aur errors automatically log karta hai."

**Do Tables Point**:
- "Do tables (`Products` aur `Ratings`) ka one-to-many relation hai. `Include(p => p.Ratings)` se ratings fetch kiye, aur `AverageRating` calculate kiya."

---

### **Next Steps**
- **Add Files**:
  - `Models/Rating.cs`
  - `DTOs/RatingRequest.cs`, `RatingResponse.cs`
  - `Interfaces/IRatingRepository.cs`, `IRatingService.cs`
  - 