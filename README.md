

### **Requirements Recap**
1. **Rating and Review**:
   - Add `Ratings` table and APIs for submitting/fetching ratings/reviews.
   - Update `Product` model to include ratings.
2. **Stock Management**:
   - Add `Stock` field to `Product` model.
   - Subtract stock when order is placed (based on quantity).
   - Add stock back when order is canceled.
3. **Email Features**:
   - **Forgot Password**: Send email with reset link/token.
   - **Reset Password**: API to reset password using token.
   - **Email Confirmation**: Send OTP during registration for email verification.
4. **Current `Product` Model**:
   ```csharp
   public class Product
   {
       [Key]
       public int Id { get; set; }
       [Required]
       public string Name { get; set; }
       public string Type { get; set; }
       public string Category { get; set; }
       [Required]
       public decimal Price { get; set; }
       public string? Description { get; set; }
       public string? Images { get; set; }
       public string? Specifications { get; set; }
       public int OwnerId { get; set; }
       public UserProfile Owner { get; set; } = null!;
   }
   ```
5. **Constraints**:
   - Keep AutoMapper simple (only essential mappings).
   - No frontend work for now.
   - Complete all requested backend features.

---

### **Implementation Plan**
1. Update database schema (`Product`, `Rating`, `Order`, `OrderItem`).
2. Implement rating/review APIs.
3. Implement stock management in order APIs.
4. Implement email features (forgot password, reset password, email confirmation).
5. Set up minimal AutoMapper.
6. Test APIs and provide instructions.

---

### **Step 1: Update Database Schema**
Pehle **models** aur **DbContext** update karte hain to support ratings, stock, and email features.

#### **1. Update Product Model**
Add `Stock`, `AverageRating`, `ReviewCount`, and `Ratings` navigation property.

**`Models/Product.cs`**:
```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.Models
{
    public class Product
    {
        [Key]
        public int Id { get; set; }
        [Required]
        public string Name { get; set; }
        public string Type { get; set; }
        public string Category { get; set; }
        [Required]
        public decimal Price { get; set; }
        [Required]
        public int Stock { get; set; }
        public decimal AverageRating { get; set; }
        public int ReviewCount { get; set; }
        public string? Description { get; set; }
        public string? Images { get; set; }
        public string? Specifications { get; set; }
        public int OwnerId { get; set; }
        public UserProfile Owner { get; set; } = null!;
        public List<Rating> Ratings { get; set; } = new List<Rating>();
    }
}
```

#### **2. Create Rating Model**
**`Models/Rating.cs`**:
```csharp
using System;
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.Models
{
    public class Rating
    {
        [Key]
        public int Id { get; set; }
        [Required]
        public int ProductId { get; set; }
        public Product Product { get; set; } = null!;
        [Required]
        public int UserProfileId { get; set; }
        public UserProfile UserProfile { get; set; } = null!;
        [Range(1, 5)]
        public int StarRating { get; set; }
        public string? Review { get; set; }
        public DateTime CreatedAt { get; set; }
    }
}
```

#### **3. Update Order Models**
**`Models/Order.cs`**:
```csharp
using System;
using System.Collections.Generic;
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.Models
{
    public class Order
    {
        [Key]
        public int Id { get; set; }
        [Required]
        public int UserProfileId { get; set; }
        public UserProfile UserProfile { get; set; } = null!;
        public decimal TotalPrice { get; set; }
        public DateTime OrderDate { get; set; }
        public string Status { get; set; } = "Placed"; // Placed, Cancelled
        public List<OrderItem> Items { get; set; } = new List<OrderItem>();
    }
}
```

**`Models/OrderItem.cs`**:
```csharp
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.Models
{
    public class OrderItem
    {
        [Key]
        public int Id { get; set; }
        [Required]
        public int OrderId { get; set; }
        public Order Order { get; set; } = null!;
        [Required]
        public int ProductId { get; set; }
        public Product Product { get; set; } = null!;
        public string ProductName { get; set; }
        public decimal Price { get; set; }
        public int Quantity { get; set; }
    }
}
```

#### **4. Update UserProfile Model**
Add `Otp` and `EmailConfirmed` for email confirmation.

**`Models/UserProfile.cs`**:
```csharp
using Microsoft.AspNetCore.Identity;

namespace EShoppingZone.Models
{
    public class UserProfile : IdentityUser<int>
    {
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public string? Otp { get; set; }
        public bool EmailConfirmed { get; set; }
    }
}
```

#### **5. Update DbContext**
**`Context/EShoppingZoneDbContext.cs`**:
```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using EShoppingZone.Models;

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
        public DbSet<Rating> Ratings { get; set; }

        protected override void OnModelCreating(ModelBuilder builder)
        {
            base.OnModelCreating(builder);

            builder.Entity<Rating>()
                .HasOne(r => r.Product)
                .WithMany(p => p.Ratings)
                .HasForeignKey(r => r.ProductId);

            builder.Entity<OrderItem>()
                .HasOne(oi => oi.Order)
                .WithMany(o => o.Items)
                .HasForeignKey(oi => oi.OrderId);
        }
    }
}
```

#### **6. Create Migration**
```bash
dotnet ef migrations add AddStockRatingsAndEmailConfirmation
dotnet ef database update
```

**Action**:
- Update `Product.cs`, `Rating.cs`, `Order.cs`, `OrderItem.cs`, `UserProfile.cs`, `EShoppingZoneDbContext.cs`.
- Run migrations and verify database schema.

---

### **Step 2: Implement Rating and Review APIs**
APIs for submitting and fetching ratings/reviews.

#### **1. Create DTOs**
**`DTOs/RatingRequest.cs`**:
```csharp
namespace EShoppingZone.DTOs
{
    public class RatingRequest
    {
        public int ProductId { get; set; }
        public int StarRating { get; set; }
        public string? Review { get; set; }
    }
}
```

**`DTOs/RatingResponse.cs`**:
```csharp
using System;

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

#### **2. Create Repository**
**`Repositories/IRatingRepository.cs`**:
```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using EShoppingZone.Models;

namespace EShoppingZone.Repositories
{
    public interface IRatingRepository
    {
        Task<Rating> GetRatingAsync(int productId, int userProfileId);
        Task CreateRatingAsync(Rating rating);
        Task<List<Rating>> GetRatingsByProductAsync(int productId);
        Task<Product> GetProductAsync(int productId);
        Task UpdateProductAsync(Product product);
    }
}
```

**`Repositories/RatingRepository.cs`**:
```csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using EShoppingZone.Context;
using EShoppingZone.Models;

namespace EShoppingZone.Repositories
{
    public class RatingRepository : IRatingRepository
    {
        private readonly EShoppingZoneDbContext _context;

        public RatingRepository(EShoppingZoneDbContext context)
        {
            _context = context;
        }

        public async Task<Rating> GetRatingAsync(int productId, int userProfileId)
        {
            return await _context.Ratings
                .FirstOrDefaultAsync(r => r.ProductId == productId && r.UserProfileId == userProfileId);
        }

        public async Task CreateRatingAsync(Rating rating)
        {
            await _context.Ratings.AddAsync(rating);
            await _context.SaveChangesAsync();
        }

        public async Task<List<Rating>> GetRatingsByProductAsync(int productId)
        {
            return await _context.Ratings
                .Where(r => r.ProductId == productId)
                .ToListAsync();
        }

        public async Task<Product> GetProductAsync(int productId)
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

#### **3. Create Service**
**`Services/IRatingService.cs`**:
```csharp
using System.Threading.Tasks;
using EShoppingZone.DTOs;

namespace EShoppingZone.Services
{
    public interface IRatingService
    {
        Task<ResponseDTO<RatingResponse>> AddRatingAsync(int profileId, RatingRequest ratingRequest);
        Task<ResponseDTO<RatingResponse[]>> GetRatingsByProductAsync(int productId);
    }
}
```

**`Services/RatingService.cs`**:
```csharp
using AutoMapper;
using System;
using System.Linq;
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using EShoppingZone.Models;
using EShoppingZone.Repositories;

namespace EShoppingZone.Services
{
    public class RatingService : IRatingService
    {
        private readonly IRatingRepository _repository;
        private readonly IMapper _mapper;

        public RatingService(IRatingRepository repository, IMapper mapper)
        {
            _repository = repository;
            _mapper = mapper;
        }

        public async Task<ResponseDTO<RatingResponse>> AddRatingAsync(int profileId, RatingRequest ratingRequest)
        {
            if (ratingRequest.StarRating < 1 || ratingRequest.StarRating > 5)
            {
                return new ResponseDTO<RatingResponse>
                {
                    Success = false,
                    Message = "Star rating must be between 1 and 5."
                };
            }

            var product = await _repository.GetProductAsync(ratingRequest.ProductId);
            if (product == null)
            {
                return new ResponseDTO<RatingResponse>
                {
                    Success = false,
                    Message = "Product not found."
                };
            }

            var existingRating = await _repository.GetRatingAsync(ratingRequest.ProductId, profileId);
            if (existingRating != null)
            {
                return new ResponseDTO<RatingResponse>
                {
                    Success = false,
                    Message = "You have already rated this product."
                };
            }

            var rating = new Rating
            {
                ProductId = ratingRequest.ProductId,
                UserProfileId = profileId,
                StarRating = ratingRequest.StarRating,
                Review = ratingRequest.Review,
                CreatedAt = DateTime.UtcNow
            };

            await _repository.CreateRatingAsync(rating);

            product.Ratings.Add(rating);
            product.AverageRating = product.Ratings.Any() ? product.Ratings.Average(r => r.StarRating) : 0;
            product.ReviewCount = product.Ratings.Count;
            await _repository.UpdateProductAsync(product);

            var response = _mapper.Map<RatingResponse>(rating);
            return new ResponseDTO<RatingResponse>
            {
                Success = true,
                Message = "Rating submitted successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<RatingResponse[]>> GetRatingsByProductAsync(int productId)
        {
            var ratings = await _repository.GetRatingsByProductAsync(productId);
            var response = _mapper.Map<RatingResponse[]>(ratings);
            return new ResponseDTO<RatingResponse[]>
            {
                Success = true,
                Message = "Ratings retrieved successfully.",
                Data = response
            };
        }
    }
}
```

#### **4. Create Controller**
**`Controllers/RatingController.cs`**:
```csharp
using System.Security.Claims;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using EShoppingZone.DTOs;
using EShoppingZone.Services;

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
            var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? throw new UnauthorizedAccessException());
            var response = await _service.AddRatingAsync(profileId, ratingRequest);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }

        [HttpGet("product/{productId}")]
        public async Task<IActionResult> GetRatingsByProduct(int productId)
        {
            var response = await _service.GetRatingsByProductAsync(productId);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }
    }
}
```

---

### **Step 3: Implement Stock Management**
Stock management in **Order** APIs (place and cancel).

#### **1. Create Order DTOs**
**`DTOs/OrderResponse.cs`**:
```csharp
using System;
using System.Collections.Generic;

namespace EShoppingZone.DTOs
{
    public class OrderResponse
    {
        public int Id { get; set; }
        public decimal TotalPrice { get; set; }
        public DateTime OrderDate { get; set; }
        public string Status { get; set; }
        public List<OrderItemResponse> Items { get; set; }
    }

    public class OrderItemResponse
    {
        public int Id { get; set; }
        public int ProductId { get; set; }
        public string ProductName { get; set; }
        public decimal Price { get; set; }
        public int Quantity { get; set; }
    }
}
```

#### **2. Update Order Repository**
**`Repositories/IOrderRepository.cs`**:
```csharp
using System.Threading.Tasks;
using EShoppingZone.Models;

namespace EShoppingZone.Repositories
{
    public interface IOrderRepository
    {
        Task CreateOrderAsync(Order order);
        Task<Order> GetOrderAsync(int orderId);
        Task UpdateOrderAsync(Order order);
    }
}
```

**`Repositories/OrderRepository.cs`**:
```csharp
using System.Threading.Tasks;
using Microsoft.EntityFrameworkCore;
using EShoppingZone.Context;
using EShoppingZone.Models;

namespace EShoppingZone.Repositories
{
    public class OrderRepository : IOrderRepository
    {
        private readonly EShoppingZoneDbContext _context;

        public OrderRepository(EShoppingZoneDbContext context)
        {
            _context = context;
        }

        public async Task CreateOrderAsync(Order order)
        {
            await _context.Orders.AddAsync(order);
            await _context.SaveChangesAsync();
        }

        public async Task<Order> GetOrderAsync(int orderId)
        {
            return await _context.Orders
                .Include(o => o.Items)
                .FirstOrDefaultAsync(o => o.Id == orderId);
        }

        public async Task UpdateOrderAsync(Order order)
        {
            _context.Orders.Update(order);
            await _context.SaveChangesAsync();
        }
    }
}
```

#### **3. Update Order Service**
**`Services/IOrderService.cs`**:
```csharp
using System.Threading.Tasks;
using EShoppingZone.DTOs;

namespace EShoppingZone.Services
{
    public interface IOrderService
    {
        Task<ResponseDTO<OrderResponse>> PlaceOrderAsync(int profileId);
        Task<ResponseDTO<OrderResponse>> CancelOrderAsync(int profileId, int orderId);
    }
}
```

**`Services/OrderService.cs`**:
```csharp
using AutoMapper;
using System;
using System.Linq;
using System.Threading.Tasks;
using EShoppingZone.Context;
using EShoppingZone.DTOs;
using EShoppingZone.Models;
using EShoppingZone.Repositories;

namespace EShoppingZone.Services
{
    public class OrderService : IOrderService
    {
        private readonly IOrderRepository _repository;
        private readonly ICartRepository _cartRepository;
        private readonly EShoppingZoneDbContext _context;
        private readonly IMapper _mapper;

        public OrderService(IOrderRepository repository, ICartRepository cartRepository, EShoppingZoneDbContext context, IMapper mapper)
        {
            _repository = repository;
            _cartRepository = cartRepository;
            _context = context;
            _mapper = mapper;
        }

        public async Task<ResponseDTO<OrderResponse>> PlaceOrderAsync(int profileId)
        {
            var cart = await _cartRepository.GetCartAsync(profileId);
            if (cart == null || !cart.Items.Any())
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Cart is empty or not found."
                };
            }

            // Check stock
            foreach (var item in cart.Items)
            {
                var product = await _context.Products.FindAsync(item.ProductId);
                if (product == null || product.Stock < item.Quantity)
                {
                    return new ResponseDTO<OrderResponse>
                    {
                        Success = false,
                        Message = $"Insufficient stock for product: {item.ProductName}"
                    };
                }
            }

            // Create order
            var order = new Order
            {
                UserProfileId = profileId,
                TotalPrice = cart.TotalPrice,
                OrderDate = DateTime.UtcNow,
                Status = "Placed",
                Items = cart.Items.Select(item => new OrderItem
                {
                    ProductId = item.ProductId,
                    ProductName = item.ProductName,
                    Price = item.Price,
                    Quantity = item.Quantity
                }).ToList()
            };

            // Update stock
            foreach (var item in cart.Items)
            {
                var product = await _context.Products.FindAsync(item.ProductId);
                product.Stock -= item.Quantity;
                _context.Products.Update(product);
            }

            // Clear cart
            cart.Items.Clear();
            cart.TotalPrice = 0;
            await _cartRepository.UpdateCartAsync(cart);

            await _repository.CreateOrderAsync(order);
            await _context.SaveChangesAsync();

            var response = _mapper.Map<OrderResponse>(order);
            return new ResponseDTO<OrderResponse>
            {
                Success = true,
                Message = "Order placed successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<OrderResponse>> CancelOrderAsync(int profileId, int orderId)
        {
            var order = await _repository.GetOrderAsync(orderId);
            if (order == null || order.UserProfileId != profileId)
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Order not found."
                };
            }

            if (order.Status == "Cancelled")
            {
                return new ResponseDTO<OrderResponse>
                {
                    Success = false,
                    Message = "Order is already cancelled."
                };
            }

            // Restore stock
            foreach (var item in order.Items)
            {
                var product = await _context.Products.FindAsync(item.ProductId);
                product.Stock += item.Quantity;
                _context.Products.Update(product);
            }

            order.Status = "Cancelled";
            await _repository.UpdateOrderAsync(order);
            await _context.SaveChangesAsync();

            var response = _mapper.Map<OrderResponse>(order);
            return new ResponseDTO<OrderResponse>
            {
                Success = true,
                Message = "Order cancelled successfully.",
                Data = response
            };
        }
    }
}
```

#### **4. Create Order Controller**
**`Controllers/OrderController.cs`**:
```csharp
using System.Security.Claims;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using EShoppingZone.DTOs;
using EShoppingZone.Services;

namespace EShoppingZone.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class OrderController : ControllerBase
    {
        private readonly IOrderService _service;

        public OrderController(IOrderService service)
        {
            _service = service;
        }

        [HttpPost]
        [Authorize(Roles = "Customer")]
        public async Task<IActionResult> PlaceOrder()
        {
            var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? throw new UnauthorizedAccessException());
            var response = await _service.PlaceOrderAsync(profileId);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }

        [HttpPut("{orderId}/cancel")]
        [Authorize(Roles = "Customer")]
        public async Task<IActionResult> CancelOrder(int orderId)
        {
            var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier)?.Value ?? throw new UnauthorizedAccessException());
            var response = await _service.CancelOrderAsync(profileId, orderId);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }
    }
}
```

---

### **Step 4: Implement Email Features**
Email features ke liye **MailKit** use karenge.

#### **1. Install MailKit**
```bash
dotnet add package MailKit
```

#### **2. Configure Email Settings**
**`appsettings.json`**:
```json
{
  "EmailSettings": {
    "SmtpServer": "smtp.gmail.com",
    "SmtpPort": 587,
    "SenderName": "EShoppingZone",
    "SenderEmail": "your-email@gmail.com",
    "Username": "your-email@gmail.com",
    "Password": "your-app-password"
  }
}
```

**Note**:
- Generate Gmail App Password: Google Account → Security → 2-Step Verification → App Passwords → Create for "Mail".

#### **3. Create Email Service**
**`Services/IEmailService.cs`**:
```csharp
using System.Threading.Tasks;

namespace EShoppingZone.Services
{
    public interface IEmailService
    {
        Task SendEmailAsync(string toEmail, string subject, string body);
    }
}
```

**`Services/EmailService.cs`**:
```csharp
using System.Threading.Tasks;
using MailKit.Net.Smtp;
using MimeKit;
using Microsoft.Extensions.Configuration;

namespace EShoppingZone.Services
{
    public class EmailService : IEmailService
    {
        private readonly IConfiguration _configuration;

        public EmailService(IConfiguration configuration)
        {
            _configuration = configuration;
        }

        public async Task SendEmailAsync(string toEmail, string subject, string body)
        {
            var message = new MimeMessage();
            message.From.Add(new MailboxAddress(
                _configuration["EmailSettings:SenderName"],
                _configuration["EmailSettings:SenderEmail"]
            ));
            message.To.Add(new MailboxAddress("", toEmail));
            message.Subject = subject;
            message.Body = new TextPart("html") { Text = body };

            using var client = new SmtpClient();
            await client.ConnectAsync(
                _configuration["EmailSettings:SmtpServer"],
                int.Parse(_configuration["EmailSettings:SmtpPort"]),
                MailKit.Security.SecureSocketOptions.StartTls
            );
            await client.AuthenticateAsync(
                _configuration["EmailSettings:Username"],
                _configuration["EmailSettings:Password"]
            );
            await client.SendAsync(message);
            await client.DisconnectAsync(true);
        }
    }
}
```

#### **4. Create Auth DTOs**
**`DTOs/ForgotPasswordRequest.cs`**:
```csharp
namespace EShoppingZone.DTOs
{
    public class ForgotPasswordRequest
    {
        public string Email { get; set; }
    }
}
```

**`DTOs/ResetPasswordRequest.cs`**:
<xaiArtifact artifact_id="3f331683-b253-4cff-b3c2-a2e2ee3ef870" artifact_version_id="94554134-ea4e-47ff-bb44-10e28b1bd7a0" title="ResetPasswordRequest.cs" contentType="text/csharp">
namespace EShoppingZone.DTOs
{
    public class ResetPasswordRequest
    {
        public string Email { get; set; }
        public string Token { get; set; }
        public string NewPassword { get; set; }
    }
}
</xaiArtifact>

**`DTOs/ConfirmEmailRequest.cs`**:
```csharp
namespace EShoppingZone.DTOs
{
    public class ConfirmEmailRequest
    {
        public string Email { get; set; }
        public string Otp { get; set; }
    }
}
```

#### **5. Update Auth Service**
**`Services/IAuthService.cs`**:
```csharp
using System.Threading.Tasks;
using EShoppingZone.DTOs;

namespace EShoppingZone.Services
{
    public interface IAuthService
    {
        Task<ResponseDTO<string>> LoginAsync(LoginRequest loginRequest);
        Task<ResponseDTO<string>> RegisterAsync(RegisterRequest registerRequest);
        Task<ResponseDTO<string>> ForgotPasswordAsync(ForgotPasswordRequest request);
        Task<ResponseDTO<string>> ResetPasswordAsync(ResetPasswordRequest request);
        Task<ResponseDTO<string>> ConfirmEmailAsync(string email, string otp);
    }
}
```

**`Services/AuthService.cs`**:
```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.Configuration;
using Microsoft.IdentityModel.Tokens;
using System;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using System.Threading.Tasks;
using EShoppingZone.Context;
using EShoppingZone.DTOs;
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public class AuthService : IAuthService
    {
        private readonly UserManager<UserProfile> _userManager;
        private readonly SignInManager<UserProfile> _signInManager;
        private readonly IConfiguration _configuration;
        private readonly IEmailService _emailService;
        private readonly EShoppingZoneDbContext _context;

        public AuthService(
            UserManager<UserProfile> userManager,
            SignInManager<UserProfile> signInManager,
            IConfiguration configuration,
            IEmailService emailService,
            EShoppingZoneDbContext context)
        {
            _userManager = userManager;
            _signInManager = signInManager;
            _configuration = configuration;
            _emailService = emailService;
            _context = context;
        }

        public async Task<ResponseDTO<string>> LoginAsync(LoginRequest loginRequest)
        {
            var user = await _userManager.FindByEmailAsync(loginRequest.Email);
            if (user == null || !user.EmailConfirmed)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid email or email not confirmed."
                };
            }

            var result = await _signInManager.CheckPasswordSignInAsync(user, loginRequest.Password, false);
            if (!result.Succeeded)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid password."
                };
            }

            var claims = new[]
            {
                new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
                new Claim(ClaimTypes.Role, "Customer")
            };

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
            var token = new JwtSecurityToken(
                issuer: _configuration["Jwt:Issuer"],
                audience: _configuration["Jwt:Audience"],
                claims: claims,
                expires: DateTime.Now.AddDays(1),
                signingCredentials: creds
            );

            return new ResponseDTO<string>
            {
                Success = true,
                Message = "Login successful.",
                Data = new JwtSecurityTokenHandler().WriteToken(token)
            };
        }

        public async Task<ResponseDTO<string>> RegisterAsync(RegisterRequest registerRequest)
        {
            var user = new UserProfile
            {
                UserName = registerRequest.Email,
                Email = registerRequest.Email,
                FirstName = registerRequest.FirstName,
                LastName = registerRequest.LastName,
                EmailConfirmed = false
            };

            var result = await _userManager.CreateAsync(user, registerRequest.Password);
            if (!result.Succeeded)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = string.Join(", ", result.Errors.Select(e => e.Description))
                };
            }

            await _userManager.AddToRoleAsync(user, "Customer");

            var otp = new Random().Next(100000, 999999).ToString();
            user.Otp = otp;
            await _userManager.UpdateAsync(user);

            var body = $"<p>Your OTP for email verification is: <strong>{otp}</strong></p>";
            await _emailService.SendEmailAsync(user.Email, "Verify Your Email", body);

            return new ResponseDTO<string>
            {
                Success = true,
                Message = "Registration successful. Please verify your email with the OTP sent."
            };
        }

        public async Task<ResponseDTO<string>> ForgotPasswordAsync(ForgotPasswordRequest request)
        {
            var user = await _userManager.FindByEmailAsync(request.Email);
            if (user == null)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not found."
                };
            }

            var token = await _userManager.GeneratePasswordResetTokenAsync(user);
            var resetLink = $"https://your-frontend-url/reset-password?email={request.Email}&token={Uri.EscapeDataString(token)}";
            var body = $"<p>Click the link to reset your password: <a href='{resetLink}'>Reset Password</a></p>";

            await _emailService.SendEmailAsync(request.Email, "Reset Your Password", body);

            return new ResponseDTO<string>
            {
                Success = true,
                Message = "Password reset link sent to your email."
            };
        }

        public async Task<ResponseDTO<string>> ResetPasswordAsync(ResetPasswordRequest request)
        {
            var user = await _userManager.FindByEmailAsync(request.Email);
            if (user == null)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not found."
                };
            }

            var result = await _userManager.ResetPasswordAsync(user, request.Token, request.NewPassword);
            if (!result.Succeeded)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = string.Join(", ", result.Errors.Select(e => e.Description))
                };
            }

            return new ResponseDTO<string>
            {
                Success = true,
                Message = "Password reset successfully."
            };
        }

        public async Task<ResponseDTO<string>> ConfirmEmailAsync(string email, string otp)
        {
            var user = await _userManager.FindByEmailAsync(email);
            if (user == null)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = "User not found."
                };
            }

            if (user.Otp != otp)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid OTP."
                };
            }

            user.EmailConfirmed = true;
            user.Otp = null;
            await _userManager.UpdateAsync(user);

            return new ResponseDTO<string>
            {
                Success = true,
                Message = "Email confirmed successfully."
            };
        }
    }
}
```

#### **6. Update Auth Controller**
**`Controllers/AuthController.cs`**:
```csharp
using System.Threading.Tasks;
using Microsoft.AspNetCore.Mvc;
using EShoppingZone.DTOs;
using EShoppingZone.Services;

namespace EShoppingZone.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class AuthController : ControllerBase
    {
        private readonly IAuthService _service;

        public AuthController(IAuthService service)
        {
            _service = service;
        }

        [HttpPost("login")]
        public async Task<IActionResult> Login([FromBody] LoginRequest loginRequest)
        {
            var response = await _service.LoginAsync(loginRequest);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }

        [HttpPost("register")]
        public async Task<IActionResult> Register([FromBody] RegisterRequest registerRequest)
        {
            var response = await _service.RegisterAsync(registerRequest);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }

        [HttpPost("forgot-password")]
        public async Task<IActionResult> ForgotPassword([FromBody] ForgotPasswordRequest request)
        {
            var response = await _service.ForgotPasswordAsync(request);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }

        [HttpPost("reset-password")]
        public async Task<IActionResult> ResetPassword([FromBody] ResetPasswordRequest request)
        {
            var response = await _service.ResetPasswordAsync(request);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }

        [HttpPost("confirm-email")]
        public async Task<IActionResult> ConfirmEmail([FromBody] ConfirmEmailRequest request)
        {
            var response = await _service.ConfirmEmailAsync(request.Email, request.Otp);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }
    }
}
```

---

### **Step 5: Minimal AutoMapper Setup**
AutoMapper ko simple rakhte hain.

#### **1. Install AutoMapper**
```bash
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
```

#### **2. Create Mapping Profile**
**`MappingProfile.cs`**:
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
            CreateMap<Rating, RatingResponse>();
            CreateMap<Order, OrderResponse>();
            CreateMap<OrderItem, OrderItemResponse>();
        }
    }
}
```

#### **3. Register AutoMapper**
**`Program.cs`** (update):
```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using System.Text;
using EShoppingZone.Context;
using EShoppingZone.Models;
using EShoppingZone.Repositories;
using EShoppingZone.Services;
using AutoMapper;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// DbContext
builder.Services.AddDbContext<EShoppingZoneDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Identity
builder.Services.AddIdentity<UserProfile, IdentityRole<int>>()
    .AddEntityFrameworkStores<EShoppingZoneDbContext>()
    .AddDefaultTokenProviders();

// JWT Authentication
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
}).AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],
        ValidAudience = builder.Configuration["Jwt:Audience"],
        IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]))
    };
});

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", builder =>
    {
        builder.AllowAnyOrigin()
               .AllowAnyHeader()
               .AllowAnyMethod();
    });
});

// Repositories
builder.Services.AddScoped<ICartRepository, CartRepository>();
builder.Services.AddScoped<IRatingRepository, RatingRepository>();
builder.Services.AddScoped<IOrderRepository, OrderRepository>();

// Services
builder.Services.AddScoped<IAuthService, AuthService>();
builder.Services.AddScoped<IRatingService, RatingService>();
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddScoped<IEmailService, EmailService>();

// AutoMapper
builder.Services.AddAutoMapper(typeof(MappingProfile));

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
    app.UseDeveloperExceptionPage();
}

app.UseHttpsRedirection();
app.UseCors("AllowAll");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

### **Step 6: Test APIs**
Run backend:
```bash
dotnet run
```

**Test Cases** (use Postman/Swagger):
1. **Register**:
   - `POST /api/auth/register`:
     ```json
     {
         "firstName": "John",
         "lastName": "Doe",
         "email": "john.doe@example.com",
         "password": "Password123!"
     }
     ```
   - Check email for OTP.

2. **Confirm Email**:
   - `POST /api/auth/confirm-email`:
     ```json
     {
         "email": "john.doe@example.com",
         "otp": "123456"
     }
     ```

3. **Login**:
   - `POST /api/auth/login`:
     ```json
     {
         "email": "john.doe@example.com",
         "password": "Password123!"
     }
     ```
   - Get JWT token.

4. **Forgot Password**:
   - `POST /api/auth/forgot-password`:
     ```json
     {
         "email": "john.doe@example.com"
     }
     ```
   - Check email for reset link.

5. **Reset Password**:
   - `POST /api/auth/reset-password`:
     ```json
     {
         "email": "john.doe@example.com",
         "token": "<received-token>",
         "newPassword": "NewPassword123!"
     }
     ```

6. **Add Rating**:
   - `POST /api/Rating` (with JWT):
     ```json
     {
         "productId": 1,
         "starRating": 4,
         "review": "Great product!"
     }
     ```

7. **Get Ratings**:
   - `GET /api/Rating/product/1`

8. **Place Order**:
   - `POST /api/Order` (with JWT)
   - Check `Products` table for updated stock.

9. **Cancel Order**:
   - `PUT /api/Order/1/cancel` (with JWT)
   - Check `Products` table for restored stock.

**Action**:
- Test all APIs.
- Check backend logs (`Logs/log-YYYYMMDD.txt`) for errors.
- Verify database changes (e.g., stock, ratings, orders).

---

### **Files Updated/Created**
- **Models**: `Product.cs`, `Rating.cs`, `Order.cs`, `OrderItem.cs`, `UserProfile.cs`
- **Context**: `EShoppingZoneDbContext.cs`
- **DTOs**: `RatingRequest.cs`, `RatingResponse.cs`, `OrderResponse.cs`, `ForgotPasswordRequest.cs`, `ResetPasswordRequest.cs`, `ConfirmEmailRequest.cs`
- **Repositories**: `IRatingRepository.cs`, `RatingRepository.cs`, `IOrderRepository.cs`, `OrderRepository.cs`
- **Services**: `IRatingService.cs`, `RatingService.cs`, `IOrderService.cs`, `OrderService.cs`, `IEmailService.cs`, `EmailService.cs`, `IAuthService.cs`, `AuthService.cs`
- **Controllers**: `RatingController.cs`, `OrderController.cs`, `AuthController.cs`
- **Others**: `MappingProfile.cs`, `Program.cs`, `appsettings.json`

---

### **Next Steps**
1. **Implement Changes**:
   - Add all provided files to your project.
   - Update `appsettings.json` with email settings.
   - Run migrations (`dotnet ef migrations add`, `dotnet ef database update`).
2. **Test Thoroughly**:
   - Test all APIs in Postman/Swagger.
   - Verify stock updates, ratings, and email delivery.
   - Share any errors (exact message, stack trace, logs).
3. **If Frontend Needed Later**:
   - Bol dena, mai Angular setup ke liye guide dunga.
4. **Mentor Prep**:
   - **Rating**: "Maine `Ratings` table add kiya, one-to-many relation with `Products`. APIs banaye submit aur fetch ratings ke liye."
   - **Stock**: "Order placement pe stock subtract hota hai, aur cancellation pe wapas add. `OrderService` mein logic handle kiya."
   - **Email**: "MailKit use kiya for OTP aur password reset. OTP registration ke time bheja jata hai."
   - **AutoMapper**: "Simple mappings banaye for `Rating` aur `Order` DTOs."
