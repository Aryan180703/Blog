
### **Solution Overview**
- **Existing**: Welcome email already setup in `AuthService` via `EmailService` (MailKit).
- **New**:
  - **Route**: `POST /api/auth/generate-otp` to generate OTP, store in-memory (`ConcurrentDictionary`), and send via email.
  - **Method**: `SendOtpEmail` in `EmailService` for OTP emails, keeping other emails (e.g., welcome) separate.
  - **Flow**: User email bhejta hai, OTP generate hota hai, in-memory store hota hai (10-min expiry), aur OTP email bheja jata hai. Verification aur registration same rahega.
- **Files**: Update `EmailService`, `AuthService`, `AuthController`, aur baaki files as needed.

---

### **Complete Code**

#### **1. UserProfile Model**
**`Models/UserProfile.cs`**:

```csharp
using Microsoft.AspNetCore.Identity;

namespace EShoppingZone.Models
{
    public class UserProfile : IdentityUser<int>
    {
        public string FirstName { get; set; }
        public string LastName { get; set; }
    }
}
```

#### **2. DTOs**
**`DTOs/RegisterRequest.cs`**:

```csharp
namespace EShoppingZone.DTOs
{
    public class RegisterRequest
    {
        public string FirstName { get; set; }
        public string LastName { get; set; }
        public string Email { get; set; }
        public string Password { get; set; }
    }
}
```

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

**`DTOs/GenerateOtpRequest.cs`** (new, for OTP generation):

```csharp
namespace EShoppingZone.DTOs
{
    public class GenerateOtpRequest
    {
        public string Email { get; set; }
    }
}
```

**`DTOs/ResponseDTO.cs`**:

```csharp
namespace EShoppingZone.DTOs
{
    public class ResponseDTO<T>
    {
        public bool Success { get; set; }
        public string Message { get; set; }
        public T Data { get; set; }
    }
}
```

#### **3. In-Memory OTP Store**
**`Services/InMemoryOtpStore.cs`**:

```csharp
using System;
using System.Collections.Concurrent;

namespace EShoppingZone.Services
{
    public static class InMemoryOtpStore
    {
        private static readonly ConcurrentDictionary<string, (string Otp, DateTime ExpiresAt)> _otpStore = new();

        public static void StoreOtp(string email, string otp, DateTime expiresAt)
        {
            _otpStore[email] = (otp, expiresAt);
        }

        public static (string Otp, DateTime ExpiresAt)? GetOtp(string email)
        {
            return _otpStore.TryGetValue(email, out var otpData) ? otpData : null;
        }

        public static void RemoveOtp(string email)
        {
            _otpStore.TryRemove(email, out _);
        }
    }
}
```

#### **4. Email Service**
**`Services/IEmailService.cs`** (updated with `SendOtpEmail`):

```csharp
using System.Threading.Tasks;

namespace EShoppingZone.Services
{
    public interface IEmailService
    {
        Task SendEmailAsync(string toEmail, string subject, string body);
        Task SendOtpEmailAsync(string toEmail, string otp);
    }
}
```

**`Services/EmailService.cs`** (added `SendOtpEmailAsync`):

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

        public async Task SendOtpEmailAsync(string toEmail, string otp)
        {
            var body = $"<p>Your OTP for email verification is: <strong>{otp}</strong></p>";
            await SendEmailAsync(toEmail, "Verify Your Email", body);
        }
    }
}
```

**Note**: `SendOtpEmailAsync` reuses `SendEmailAsync` for OTP emails, keeping code DRY. You can still use `SendEmailAsync` for welcome emails or others.

#### **5. Auth Service**
**`Services/IAuthService.cs`** (added `GenerateOtpAsync`):

```csharp
using System.Threading.Tasks;
using EShoppingZone.DTOs;

namespace EShoppingZone.Services
{
    public interface IAuthService
    {
        Task<ResponseDTO<string>> RegisterAsync(RegisterRequest registerRequest);
        Task<ResponseDTO<string>> ConfirmEmailAsync(string email, string otp);
        Task<ResponseDTO<string>> GenerateOtpAsync(string email);
    }
}
```

**`Services/AuthService.cs`** (full, with new `GenerateOtpAsync`):

```csharp
using Microsoft.AspNetCore.Identity;
using System;
using System.Linq;
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public class AuthService : IAuthService
    {
        private readonly UserManager<UserProfile> _userManager;
        private readonly IEmailService _emailService;

        public AuthService(
            UserManager<UserProfile> userManager,
            IEmailService emailService)
        {
            _userManager = userManager;
            _emailService = emailService;
        }

        public async Task<ResponseDTO<string>> RegisterAsync(RegisterRequest registerRequest)
        {
            var user = new UserProfile
            {
                UserName = registerRequest.Email,
                Email = registerRequest.Email,
                FirstName = registerRequest.FirstName,
                LastName = registerRequest.LastName
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
            InMemoryOtpStore.StoreOtp(user.Email, otp, DateTime.UtcNow.AddMinutes(10));
            await _emailService.SendOtpEmailAsync(user.Email, otp);

            // Assuming you send welcome email here (as you mentioned)
            await _emailService.SendEmailAsync(user.Email, "Welcome to EShoppingZone", 
                "<p>Welcome, thanks for joining!</p>");

            return new ResponseDTO<string>
            {
                Success = true,
                Message = "Registration successful. Please verify your email with the OTP sent."
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

            var otpData = InMemoryOtpStore.GetOtp(email);
            if (otpData == null || otpData.Value.Otp != otp || otpData.Value.ExpiresAt < DateTime.UtcNow)
            {
                return new ResponseDTO<string>
                {
                    Success = false,
                    Message = "Invalid or expired OTP."
                };
            }

            user.EmailConfirmed = true;
            await _userManager.UpdateAsync(user);
            InMemoryOtpStore.RemoveOtp(email);

            return new ResponseDTO<string>
            {
                Success = true,
                Message = "Email confirmed successfully."
            };
        }

        public async Task<ResponseDTO<string>> GenerateOtpAsync(string email)
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

            var otp = new Random().Next(100000, 999999).ToString();
            InMemoryOtpStore.StoreOtp(email, otp, DateTime.UtcNow.AddMinutes(10));
            await _emailService.SendOtpEmailAsync(email, otp);

            return new ResponseDTO<string>
            {
                Success = true,
                Message = "OTP generated and sent to your email."
            };
        }
    }
}
```

**Notes**:
- `RegisterAsync`: Uses `SendOtpEmailAsync` for OTP and `SendEmailAsync` for welcome email (as you mentioned).
- `GenerateOtpAsync`: New method for generating and sending OTP for existing users.
- `ConfirmEmailAsync`: Same as before, checks OTP in-memory.

#### **6. Auth Controller**
**`Controllers/AuthController.cs`** (added `generate-otp` route):

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

        [HttpPost("generate-otp")]
        public async Task<IActionResult> GenerateOtp([FromBody] GenerateOtpRequest request)
        {
            var response = await _service.GenerateOtpAsync(request.Email);
            if (response.Success)
            {
                return Ok(response);
            }
            return BadRequest(response);
        }
    }
}
```

#### **7. DbContext**
**`Context/EShoppingZoneDbContext.cs`** (no `OtpToken`):

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
        // Add other DbSets (e.g., Products, Ratings) as needed
    }
}
```

#### **8. Program.cs**
**`Program.cs`**:

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using EShoppingZone.Context;
using EShoppingZone.Models;
using EShoppingZone.Services;

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

// Services
builder.Services.AddScoped<IAuthService, AuthService>();
builder.Services.AddScoped<IEmailService, EmailService>();

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", builder =>
    {
        builder.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod();
    });
});

var app = builder.Build();

// Configure pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseCors("AllowAll");
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

#### **9. appsettings.json**
**`appsettings.json`**:

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=your_server;Database=EShoppingZone;Trusted_Connection=True;"
  },
  "EmailSettings": {
    "SmtpServer": "smtp.gmail.com",
    "SmtpPort": "587",
    "SenderName": "EShoppingZone",
    "SenderEmail": "your-email@gmail.com",
    "Username": "your-email@gmail.com",
    "Password": "your-app-password"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

**Note**: Replace `your-email@gmail.com` and `your-app-password` with Gmail credentials. Generate App Password from Google Account → Security → 2-Step Verification → App Passwords → Create for "Mail".

---

### **Implementation Steps**
1. **Install Packages**:
   ```bash
   dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
   dotnet add package Microsoft.EntityFrameworkCore.SqlServer
   dotnet add package MailKit
   ```

2. **Add/Update Files**:
   - `Models/UserProfile.cs`
   - `DTOs/RegisterRequest.cs`, `ConfirmEmailRequest.cs`, `GenerateOtpRequest.cs`, `ResponseDTO.cs`
   - `Services/InMemoryOtpStore.cs`, `IEmailService.cs`, `EmailService.cs`
   - `Services/IAuthService.cs`, `AuthService.cs`
   - `Controllers/AuthController.cs`
   - `Context/EShoppingZoneDbContext.cs`
   - `Program.cs`, `appsettings.json`

3. **Setup Database**:
   - Update `appsettings.json` with SQL Server connection string.
   - Run migrations:
     ```bash
     dotnet ef migrations add InitialCreate
     dotnet ef database update
     ```

4. **Test APIs**:
   - Run app:
     ```bash
     dotnet run
     ```
   - **Register** (sends OTP + welcome email):
     ```bash
     POST /api/auth/register
     ```
     ```json
     {
         "firstName": "John",
         "lastName": "Doe",
         "email": "john.doe@example.com",
         "password": "Password123!"
     }
     ```
     - Check email for OTP (e.g., "123456") and welcome email.
   - **Generate OTP** (for re-sending OTP):
     ```bash
     POST /api/auth/generate-otp
     ```
     ```json
     {
         "email": "john.doe@example.com"
     }
     ```
     - Check email for new OTP.
   - **Verify OTP**:
     ```bash
     POST /api/auth/confirm-email
     ```
     ```json
     {
         "email": "john.doe@example.com",
         "otp": "123456"
     }
     ```
     - Response:
       ```json
       {
           "success": true,
           "message": "Email confirmed successfully.",
           "data": null
       }
       ```

5. **Verify Database**:
   - Check `AspNetUsers`:
     ```sql
     SELECT Email, EmailConfirmed FROM AspNetUsers WHERE Email = 'john.doe@example.com';
     ```
     - `EmailConfirmed` should be `true` after verification.

---

### **Troubleshooting**
- **Email Not Received**:
  - Verify `appsettings.json` Gmail App Password.
  - Check spam/junk folder.
  - Add logging in `EmailService`:
    ```csharp
    Console.WriteLine($"Sending email to {toEmail}");
    ```
- **Invalid/Expired OTP**:
  - Enter OTP within 10 minutes.
  - Re-generate OTP via `POST /api/auth/generate-otp` if server restarted (in-memory clears).
  - Check email case-sensitivity.
- **Errors**:
  - Share **exact error message** and **logs** (`Logs/log-YYYYMMDD.txt`).
  - Confirm `UserProfile` inherits `IdentityUser<int>`.

---

### **Mentor Prep**
- **What You Did**: "Maine OTP verification banaya with in-memory storage. Registration pe OTP aur welcome email bheja using MailKit. Ek naya `POST /api/auth/generate-otp` route banaya jo OTP generate karta hai, `ConcurrentDictionary` mein store karta hai (10-min expiry), aur `SendOtpEmailAsync` method se email bhejta hai. Verification pe `EmailConfirmed` true hota hai."
- **Key Points**: Minimal code, reused `EmailService` for OTP and welcome emails, in-memory for OTP to avoid database.
 le! Fatafat bol agar aur kuch chahiye! 😅