
### Order CRUD Implementation

#### 1. **Models**
- **Order Model** (`Models/Order.cs`):
  ```csharp
  using System;
  using System.Collections.Generic;

  namespace EShoppingZone.Models
  {
      public class Order
      {
          public int Id { get; set; }
          public int CustomerId { get; set; }
          public int AddressId { get; set; }
          public decimal TotalPrice { get; set; }
          public string Status { get; set; } = "Placed";
          public DateTime OrderDate { get; set; }
          public UserProfile Customer { get; set; } = null!;
          public Address Address { get; set; } = null!;
          public List<OrderItem> Items { get; set; } = new();
      }
  }
  ```

- **OrderItem Model** (`Models/OrderItem.cs`):
  ```csharp
  namespace EShoppingZone.Models
  {
      public class OrderItem
      {
          public int Id { get; set; }
          public int OrderId { get; set; }
          public int ProductId { get; set; }
          public string ProductName { get; set; } = string.Empty;
          public decimal Price { get; set; }
          public int Quantity { get; set; }
          public Order Order { get; set; } = null!;
          public Product Product { get; set; } = null!;
      }
  }
  ```

#### 2. **DTOs**
- **OrderRequest** (`DTOs/Order/OrderRequest.cs`):
  ```csharp
  using System.ComponentModel.DataAnnotations;

  namespace EShoppingZone.DTOs.Order
  {
      public class OrderRequest
      {
          [Required(ErrorMessage = "Cart ID is required")]
          public int CartId { get; set; }

          [Required(ErrorMessage = "Address ID is required")]
          public int AddressId { get; set; }
      }
  }
  ```

- **OrderResponse** (`DTOs/Order/OrderResponse.cs`):
  ```csharp
  using System;
  using System.Collections.Generic;

  namespace EShoppingZone.DTOs.Order
  {
      public class OrderResponse
      {
          public int Id { get; set; }
          public int CustomerId { get; set; }
          public int AddressId { get; set; }
          public decimal TotalPrice { get; set; }
          public string Status { get; set; } = string.Empty;
          public DateTime OrderDate { get; set; }
          public List<OrderItemResponse> Items { get; set; } = new();
      }
  }
  ```

- **OrderItemResponse** (`DTOs/Order/OrderItemResponse.cs`):
  ```csharp
  namespace EShoppingZone.DTOs.Order
  {
      public class OrderItemResponse
      {
          public int Id { get; set; }
          public int ProductId { get; set; }
          public string ProductName { get; set; } = string.Empty;
          public decimal Price { get; set; }
          public int Quantity { get; set; }
      }
  }
  ```

- **UpdateOrderStatusRequest** (`DTOs/Order/UpdateOrderStatusRequest.cs`):
  ```csharp
  using System.ComponentModel.DataAnnotations;

  namespace EShoppingZone.DTOs.Order
  {
      public class UpdateOrderStatusRequest
      {
          [Required(ErrorMessage = "Status is required")]
          public string Status { get; set; } = string.Empty;
      }
  }
  ```

#### 3. **DbContext Configuration**
- Update `EShoppingZoneDbContext.cs` (`Context/EShoppingZoneDbContext.cs`):
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

          public DbSet<Address> Addresses { get; set; }
          public DbSet<Product> Products { get; set; }
          public DbSet<Cart> Carts { get; set; }
          public DbSet<CartItem> CartItems { get; set; }
          public DbSet<Order> Orders { get; set; }
          public DbSet<OrderItem> OrderItems { get; set; }

          protected override void OnModelCreating(ModelBuilder builder)
          {
              base.OnModelCreating(builder);

              // UserProfile
              builder.Entity<UserProfile>().ToTable("UserProfiles");

              // Address
              builder.Entity<Address>()
                  .HasOne(a => a.UserProfile)
                  .WithMany(up => up.Addresses)
                  .HasForeignKey(a => a.UserProfileId);

              // Product
              builder.Entity<Product>().ToTable("Products");
              builder.Entity<Product>()
                  .HasOne(p => p.Owner)
                  .WithMany()
                  .HasForeignKey(p => p.OwnerId)
                  .OnDelete(DeleteBehavior.Restrict);

              // Cart
              builder.Entity<Cart>().ToTable("Carts");
              builder.Entity<Cart>()
                  .HasOne(c => c.UserProfile)
                  .WithMany()
                  .HasForeignKey(c => c.UserProfileId)
                  .OnDelete(DeleteBehavior.Cascade);

              // CartItem
              builder.Entity<CartItem>().ToTable("CartItems");
              builder.Entity<CartItem>()
                  .HasOne(ci => ci.Cart)
                  .WithMany(c => c.Items)
                  .HasForeignKey(ci => ci.CartId)
                  .OnDelete(DeleteBehavior.Cascade);
              builder.Entity<CartItem>()
                  .HasOne(ci => ci.Product)
                  .WithMany()
                  .HasForeignKey(ci => ci.ProductId)
                  .OnDelete(DeleteBehavior.Cascade);

              // Order
              builder.Entity<Order>().ToTable("Orders");
              builder.Entity<Order>()
                  .HasOne(o => o.Customer)
                  .WithMany()
                  .HasForeignKey(o => o.CustomerId)
                  .OnDelete(DeleteBehavior.Restrict);
              builder.Entity<Order>()
                  .HasOne(o => o.Address)
                  .WithMany()
                  .HasForeignKey(o => o.AddressId)
                  .OnDelete(DeleteBehavior.Restrict);

              // OrderItem
              builder.Entity<OrderItem>().ToTable("OrderItems");
              builder.Entity<OrderItem>()
                  .HasOne(oi => oi.Order)
                  .WithMany(o => o.Items)
                  .HasForeignKey(oi => oi.OrderId)
                  .OnDelete(DeleteBehavior.Cascade);
              builder.Entity<OrderItem>()
                  .HasOne(oi => oi.Product)
                  .WithMany()
                  .HasForeignKey(oi => oi.ProductId)
                  .OnDelete(DeleteBehavior.Restrict);

              // Seed Roles
              builder.Entity<IdentityRole<int>>().HasData(
                  new IdentityRole<int> { Id = 1, Name = "Customer", NormalizedName = "CUSTOMER" },
                  new IdentityRole<int> { Id = 2, Name = "Merchant", NormalizedName = "MERCHANT" },
                  new IdentityRole<int> { Id = 3, Name = "Delivery Agent", NormalizedName = "DELIVERY AGENT" }
              );
          }
      }
  }
  ```

#### 4. **Service Interface**
- In `Interfaces/IOrderService.cs`:
  ```csharp
  using System.Threading.Tasks;
  using EShoppingZone.DTOs;
  using EShoppingZone.DTOs.Order;

  namespace EShoppingZone.Interfaces
  {
      public interface IOrderService
      {
          Task<ResponseDTO<OrderResponse>> PlaceOrderAsync(int profileId, OrderRequest orderRequest);
          Task<ResponseDTO<OrderResponse>> GetOrderAsync(int profileId, int orderId);
          Task<ResponseDTO<List<OrderResponse>>> GetAllOrdersAsync(int profileId);
          Task<ResponseDTO<OrderResponse>> UpdateOrderStatusAsync(int profileId, int orderId, UpdateOrderStatusRequest updateRequest);
          Task<ResponseDTO<OrderResponse>> CancelOrderAsync(int profileId, int orderId);
      }
  }
  ```

#### 5. **Order Service**
- In `Services/OrderService.cs`:
  ```csharp
  using System;
  using System.Collections.Generic;
  using System.Linq;
  using System.Threading.Tasks;
  using AutoMapper;
  using EShoppingZone.DTOs;
  using EShoppingZone.DTOs.Order;
  using EShoppingZone.Interfaces;
  using EShoppingZone.Models;
  using Microsoft.EntityFrameworkCore;

  namespace EShoppingZone.Services
  {
      public class OrderService : IOrderService
      {
          private readonly IOrderRepository _repository;
          private readonly IMapper _mapper;

          public OrderService(IOrderRepository repository, IMapper mapper)
          {
              _repository = repository;
              _mapper = mapper;
          }

          public async Task<ResponseDTO<OrderResponse>> PlaceOrderAsync(int profileId, OrderRequest orderRequest)
          {
              var cart = await _repository.GetCartAsync(orderRequest.CartId);
              if (cart == null || cart.UserProfileId != profileId || !cart.Items.Any())
              {
                  return new ResponseDTO<OrderResponse>
                  {
                      Success = false,
                      Message = "Invalid or empty cart"
                  };
              }

              var address = await _repository.GetAddressAsync(orderRequest.AddressId);
              if (address == null || address.UserProfileId != profileId)
              {
                  return new ResponseDTO<OrderResponse>
                  {
                      Success = false,
                      Message = "Invalid address"
                  };
              }

              var order = new Order
              {
                  CustomerId = profileId,
                  AddressId = orderRequest.AddressId,
                  TotalPrice = cart.TotalPrice,
                  Status = "Placed",
                  OrderDate = DateTime.UtcNow
              };

              foreach (var item in cart.Items)
              {
                  var product = await _repository.GetProductAsync(item.ProductId);
                  if (product == null)
                  {
                      return new ResponseDTO<OrderResponse>
                      {
                          Success = false,
                          Message = $"Product {item.ProductId} not found"
                      };
                  }
                  order.Items.Add(new OrderItem
                  {
                      ProductId = item.ProductId,
                      ProductName = item.ProductName,
                      Price = item.Price,
                      Quantity = item.Quantity
                  });
              }

              var createdOrder = await _repository.CreateOrderAsync(order);
              await _repository.ClearCartAsync(cart);

              var response = await GetOrderResponseAsync(createdOrder);
              return new ResponseDTO<OrderResponse>
              {
                  Success = true,
                  Message = "Order placed successfully",
                  Data = response
              };
          }

          public async Task<ResponseDTO<OrderResponse>> GetOrderAsync(int profileId, int orderId)
          {
              var order = await _repository.GetOrderAsync(orderId);
              if (order == null || (order.CustomerId != profileId && !await _repository.IsMerchantAsync(profileId)))
              {
                  return new ResponseDTO<OrderResponse>
                  {
                      Success = false,
                      Message = "Order not found or unauthorized"
                  };
              }

              var response = await GetOrderResponseAsync(order);
              return new ResponseDTO<OrderResponse>
              {
                  Success = true,
                  Message = "Order retrieved",
                  Data = response
              };
          }

          public async Task<ResponseDTO<List<OrderResponse>>> GetAllOrdersAsync(int profileId)
          {
              var isMerchant = await _repository.IsMerchantAsync(profileId);
              var orders = await _repository.GetAllOrdersAsync(profileId, isMerchant);

              if (!orders.Any())
              {
                  return new ResponseDTO<List<OrderResponse>>
                  {
                      Success = false,
                      Message = "No orders found"
                  };
              }

              var response = new List<OrderResponse>();
              foreach (var order in orders)
              {
                  response.Add(await GetOrderResponseAsync(order));
              }

              return new ResponseDTO<List<OrderResponse>>
              {
                  Success = true,
                  Message = "Orders retrieved",
                  Data = response
              };
          }

          public async Task<ResponseDTO<OrderResponse>> UpdateOrderStatusAsync(int profileId, int orderId, UpdateOrderStatusRequest updateRequest)
          {
              var isDeliveryAgent = await _repository.IsDeliveryAgentAsync(profileId);
              if (!isDeliveryAgent)
              {
                  return new ResponseDTO<OrderResponse>
                  {
                      Success = false,
                      Message = "Unauthorized"
                  };
              }

              var order = await _repository.GetOrderAsync(orderId);
              if (order == null)
              {
                  return new ResponseDTO<OrderResponse>
                  {
                      Success = false,
                      Message = "Order not found"
                  };
              }

              var validStatuses = new[] { "Placed", "Shipped", "Delivered", "Cancelled" };
              if (!validStatuses.Contains(updateRequest.Status))
              {
                  return new ResponseDTO<OrderResponse>
                  {
                      Success = false,
                      Message = "Invalid status"
                  };
              }

              order.Status = updateRequest.Status;
              await _repository.UpdateOrderAsync(order);

              var response = await GetOrderResponseAsync(order);
              return new ResponseDTO<OrderResponse>
              {
                  Success = true,
                  Message = "Order status updated",
                  Data = response
              };
          }

          public async Task<ResponseDTO<OrderResponse>> CancelOrderAsync(int profileId, int orderId)
          {
              var order = await _repository.GetOrderAsync(orderId);
              if (order == null || order.CustomerId != profileId)
              {
                  return new ResponseDTO<OrderResponse>
                  {
                      Success = false,
                      Message = "Order not found or unauthorized"
                  };
              }

              if (order.Status != "Placed")
              {
                  return new ResponseDTO<OrderResponse>
                  {
                      Success = false,
                      Message = "Cannot cancel order, already processed"
                  };
              }

              order.Status = "Cancelled";
              await _repository.UpdateOrderAsync(order);

              var response = await GetOrderResponseAsync(order);
              return new ResponseDTO<OrderResponse>
              {
                  Success = true,
                  Message = "Order cancelled",
                  Data = response
              };
          }

          private async Task<OrderResponse> GetOrderResponseAsync(Order order)
          {
              var response = _mapper.Map<OrderResponse>(order);
              response.Items = order.Items.Select(i => _mapper.Map<OrderItemResponse>(i)).ToList();
              return response;
          }
      }
  }
  ```

#### 6. **Repository Interface**
- In `Interfaces/IOrderRepository.cs`:
  ```csharp
  using System.Collections.Generic;
  using System.Threading.Tasks;
  using EShoppingZone.Models;

  namespace EShoppingZone.Interfaces
  {
      public interface IOrderRepository
      {
          Task<Order> CreateOrderAsync(Order order);
          Task<Order> GetOrderAsync(int orderId);
          Task<List<Order>> GetAllOrdersAsync(int profileId, bool isMerchant);
          Task UpdateOrderAsync(Order order);
          Task<Cart> GetCartAsync(int cartId);
          Task<Address> GetAddressAsync(int addressId);
          Task<Product> GetProductAsync(int productId);
          Task ClearCartAsync(Cart cart);
          Task<bool> IsMerchantAsync(int profileId);
          Task<bool> IsDeliveryAgentAsync(int profileId);
      }
  }
  ```

#### 7. **Order Repository**
- In `Repositories/OrderRepository.cs`:
  ```csharp
  using System.Collections.Generic;
  using System.Linq;
  using System.Threading.Tasks;
  using EShoppingZone.Context;
  using EShoppingZone.Models;
  using Microsoft.AspNetCore.Identity;
  using Microsoft.EntityFrameworkCore;

  namespace EShoppingZone.Repositories
  {
      public class OrderRepository : IOrderRepository
      {
          private readonly EShoppingZoneDbContext _context;
          private readonly RoleManager<IdentityRole<int>> _roleManager;

          public OrderRepository(EShoppingZoneDbContext context, RoleManager<IdentityRole<int>> roleManager)
          {
              _context = context;
              _roleManager = roleManager;
          }

          public async Task<Order> CreateOrderAsync(Order order)
          {
              await _context.Orders.AddAsync(order);
              await _context.SaveChangesAsync();
              return order;
          }

          public async Task<Order> GetOrderAsync(int orderId)
          {
              return await _context.Orders
                  .Include(o => o.Items)
                  .FirstOrDefaultAsync(o => o.Id == orderId);
          }

          public async Task<List<Order>> GetAllOrdersAsync(int profileId, bool isMerchant)
          {
              if (isMerchant)
              {
                  return await _context.Orders
                      .Include(o => o.Items)
                      .ToListAsync();
              }
              return await _context.Orders
                  .Include(o => o.Items)
                  .Where(o => o.CustomerId == profileId)
                  .ToListAsync();
          }

          public async Task UpdateOrderAsync(Order order)
          {
              _context.Orders.Update(order);
              await _context.SaveChangesAsync();
          }

          public async Task<Cart> GetCartAsync(int cartId)
          {
              return await _context.Carts
                  .Include(c => c.Items)
                  .FirstOrDefaultAsync(c => c.Id == cartId);
          }

          public async Task<Address> GetAddressAsync(int addressId)
          {
              return await _context.Addresses.FindAsync(addressId);
          }

          public async Task<Product> GetProductAsync(int productId)
          {
              return await _context.Products.FindAsync(productId);
          }

          public async Task ClearCartAsync(Cart cart)
          {
              cart.Items.Clear();
              cart.TotalPrice = 0;
              _context.Carts.Update(cart);
              await _context.SaveChangesAsync();
          }

          public async Task<bool> IsMerchantAsync(int profileId)
          {
              var userRoles = await _context.UserRoles
                  .Where(ur => ur.UserId == profileId)
                  .Select(ur => ur.RoleId)
                  .ToListAsync();
              var merchantRole = await _roleManager.FindByNameAsync("Merchant");
              return userRoles.Contains(merchantRole.Id);
          }

          public async Task<bool> IsDeliveryAgentAsync(int profileId)
          {
              var userRoles = await _context.UserRoles
                  .Where(ur => ur.UserId == profileId)
                  .Select(ur => ur.RoleId)
                  .ToListAsync();
              var deliveryAgentRole = await _roleManager.FindByNameAsync("Delivery Agent");
              return userRoles.Contains(deliveryAgentRole.Id);
          }
      }
  }
  ```

#### 8. **Order Controller**
- In `Controllers/OrderController.cs`:
  ```csharp
  using System;
  using System.Security.Claims;
  using System.Threading.Tasks;
  using EShoppingZone.DTOs;
  using EShoppingZone.DTOs.Order;
  using EShoppingZone.Interfaces;
  using Microsoft.AspNetCore.Authorization;
  using Microsoft.AspNetCore.Mvc;

  namespace EShoppingZone.Controllers
  {
      [Route("api/OrderController")]
      [ApiController]
      [Authorize]
      public class OrderController : ControllerBase
      {
          private readonly IOrderService _service;

          public OrderController(IOrderService service)
          {
              _service = service;
          }

          [HttpPost]
          [Authorize(Roles = "Customer")]
          public async Task<IActionResult> PlaceOrder([FromBody] OrderRequest orderRequest)
          {
              try
              {
                  var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier).Value);
                  var response = await _service.PlaceOrderAsync(profileId, orderRequest);
                  if (response.Success)
                  {
                      return Ok(response);
                  }
                  return BadRequest(response);
              }
              catch (Exception ex)
              {
                  return BadRequest(ex.Message);
              }
          }

          [HttpGet("{orderId}")]
          [Authorize(Roles = "Customer,Merchant")]
          public async Task<IActionResult> GetOrder(int orderId)
          {
              try
              {
                  var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier).Value);
                  var response = await _service.GetOrderAsync(profileId, orderId);
                  if (response.Success)
                  {
                      return Ok(response);
                  }
                  return BadRequest(response);
              }
              catch (Exception ex)
              {
                  return BadRequest(ex.Message);
              }
          }

          [HttpGet]
          [Authorize(Roles = "Customer,Merchant")]
          public async Task<IActionResult> GetAllOrders()
          {
              try
              {
                  var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier).Value);
                  var response = await _service.GetAllOrdersAsync(profileId);
                  if (response.Success)
                  {
                      return Ok(response);
                  }
                  return BadRequest(response);
              }
              catch (Exception ex)
              {
                  return BadRequest(ex.Message);
              }
          }

          [HttpPut("{orderId}/status")]
          [Authorize(Roles = "Delivery Agent")]
          public async Task<IActionResult> UpdateOrderStatus(int orderId, [FromBody] UpdateOrderStatusRequest updateRequest)
          {
              try
              {
                  var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier).Value);
                  var response = await _service.UpdateOrderStatusAsync(profileId, orderId, updateRequest);
                  if (response.Success)
                  {
                      return Ok(response);
                  }
                  return BadRequest(response);
              }
              catch (Exception ex)
              {
                  return BadRequest(ex.Message);
              }
          }

          [HttpDelete("{orderId}")]
          [Authorize(Roles = "Customer")]
          public async Task<IActionResult> CancelOrder(int orderId)
          {
              try
              {
                  var profileId = int.Parse(User.FindFirst(ClaimTypes.NameIdentifier).Value);
                  var response = await _service.CancelOrderAsync(profileId, orderId);
                  if (response.Success)
                  {
                      return Ok(response);
                  }
                  return BadRequest(response);
              }
              catch (Exception ex)
              {
                  return BadRequest(ex.Message);
              }
          }
      }
  }
  ```

#### 9. **AutoMapper Configuration**
- Update `MappingProfile.cs` (`Profiles/MappingProfile.cs`):
  ```csharp
  using AutoMapper;
  using EShoppingZone.DTOs.Cart;
  using EShoppingZone.DTOs.Order;
  using EShoppingZone.Models;

  namespace EShoppingZone.Profiles
  {
      public class MappingProfile : Profile
      {
          public MappingProfile()
          {
              // Existing mappings (Address, Product, Cart)...
              CreateMap<Cart, CartResponse>();
              CreateMap<CartItem, CartItemResponse>();
              CreateMap<CartRequest, CartItem>();

              // Order mappings
              CreateMap<Order, OrderResponse>();
              CreateMap<OrderItem, OrderItemResponse>();
              CreateMap<OrderRequest, Order>();
          }
      }
  }
  ```
- Ensure AutoMapper is registered in `Program.cs`:
  ```csharp
  builder.Services.AddAutoMapper(typeof(MappingProfile));
  ```

#### 10. **Update Program.cs**
- Register services:
  ```csharp
  builder.Services.AddScoped<IOrderService, OrderService>();
  builder.Services.AddScoped<IOrderRepository, OrderRepository>();
  ```

#### 11. **Run Migrations**
- Generate and apply migrations:
  ```bash
  dotnet ef migrations add AddOrderTables
  dotnet ef database update
  ```

---

### Testing
1. **Run Project**:
   ```bash
   dotnet run
   ```
2. **Test APIs** (Swagger: `https://localhost:5001/swagger` or Postman):
   - **Login as Customer**: `POST /api/auth/login` to get JWT token.
   - **Place Order**: `POST /api/OrderController` (with `Bearer <token>`).
     ```json
     {
         "cartId": 1,
         "addressId": 1
     }
     ```
   - **Get Order**: `GET /api/OrderController/1` (Customer/Merchant).
   - **Get All Orders**: `GET /api/OrderController` (Customer: own, Merchant: all).
   - **Update Status**: `PUT /api/OrderController/1/status` (Delivery Agent, e.g., `{"status": "Shipped"}`).
   - **Cancel Order**: `DELETE /api/OrderController/1` (Customer, only if "Placed").
3. **Verify DB**:
   - Check `Orders` and `OrderItems` tables; `CustomerId`, `AddressId`, `Status` match karein.

---

### Notes
- **Style Match**: Code tere `AddressController`, `CartService`, `ResponseDTO` style mein hai (e.g., `profileId`, `Success/Message/Data`, repository pattern).
- **Authorization**: `[Authorize(Roles = "...")]` se Customer, Merchant, Delivery Agent ke roles control kiye.
- **User ID**: JWT se `ClaimTypes.NameIdentifier` nikal ke `profileId` set kiya.
- **Schema**: `Orders`, `OrderItems` tables follow artifact ID `a6ec9eff-ab46-4907-91aa-39e358d57b6b`.
- **Cascade**: `OrderItem.OrderId` pe `Cascade` (order delete -> items delete), `ProductId` pe `Restrict` (product delete block karega agar order mein hai).
- **Memories**: Tere preference use kiye: DTO folder (`Order/`), repository pattern, Customer/Merchant/Delivery Agent roles (April 28, 2025).


Project Structure:
EShoppingZone/
├── Controllers/
│   └── OrderController.cs
├── Services/
│   └── OrderService.cs
├── Repositories/
│   └── OrderRepository.cs
├── Models/
│   ├── Order.cs
│   └── OrderItem.cs
├── DTOs/
│   └── Order/
│       ├── OrderRequest.cs
│       ├── OrderResponse.cs
│       ├── OrderItemResponse.cs
│       └── UpdateOrderStatusRequest.cs
├── Interfaces/
│   ├── IOrderService.cs
│   └── IOrderRepository.cs
├── Context/
│   └── EShoppingZoneDbContext.cs
├── Profiles/
│   └── MappingProfile.cs

Steps to Run:
1. Add Order.cs, OrderItem.cs, DTOs (OrderRequest.cs, OrderResponse.cs, OrderItemResponse.cs, UpdateOrderStatusRequest.cs), IOrderService.cs, OrderService.cs, IOrderRepository.cs, OrderRepository.cs, OrderController.cs to respective folders.
2. Update EShoppingZoneDbContext.cs with Orders and OrderItems DbSets and OnModelCreating configuration.
3. Update MappingProfile.cs with Order mappings and ensure AutoMapper is registered in Program.cs.
4. Register IOrderService and IOrderRepository in Program.cs.
5. Run migrations: `dotnet ef migrations add AddOrderTables`, `dotnet ef database update`.
6. Run project: `dotnet run`.
7. Test APIs with Swagger (`https://localhost:5001/swagger`) or Postman, using JWT tokens for Customer, Merchant, Delivery Agent roles.


---

