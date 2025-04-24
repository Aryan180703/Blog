Thanks for the additional details! Your mentor has asked you to use **Microsoft.AspNetCore.Identity.EntityFrameworkCore** and **Microsoft.AspNetCore.Authentication.JwtBearer** for implementing JWT-based authentication in your **EShoppingZone** .NET Core WebAPI project (April 23, 2025, 16:02). Since you want to apply JWT now and have completed the Profile module (April 23, 2025, 15:11), I’ll provide a solution that integrates **ASP.NET Core Identity** with JWT for **login**, **register**, and **password reset**, replacing the custom authentication approach (April 23, 2025, 15:37). All code will be in **copyable code blocks** as requested (April 23, 2025, 15:11), and I’ll keep explanations minimal, focusing on implementation steps.

### Assumptions
- Replace custom `UserProfile` authentication with **ASP.NET Core Identity** (`IdentityUser`).
- Use `IdentityUser<int>` (integer keys to align with `ProfileId`) and extend with custom properties (e.g., `FullName`, `RoleId`).
- JWT `sub` claim holds user ID (`ProfileId`) for Profile module integration (April 23, 2025, 14:45).
- Password reset via email link (using Identity’s token system).
- Reuse existing `EShoppingZoneDbContext`, `ResponseDTO<T>` with SMD structure, and SQL Server (April 23, 2025, 12:58).
- Drop custom `PasswordResetToken` model (April 23, 2025, 15:37) as Identity handles tokens.

### Steps to Implement JWT with ASP.NET Core Identity

1. **Install Packages** (update `EShoppingZone.csproj`):
   ```xml
   <ItemGroup>
     <PackageReference Include="Microsoft.AspNetCore.Identity.EntityFrameworkCore" Version="8.0.8" />
     <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.8" />
     <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.8" />
     <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="8.0.8" />
     <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.8" />
     <PackageReference Include="Microsoft.AspNetCore.Mvc.NewtonsoftJson" Version="8.0.8" />
     <PackageReference Include="System.Net.Mail" Version="4.7.0" />
   </ItemGroup>
   ```
   Run: `dotnet restore`

2. **Update Files** (new and modified files below).

### Updated/New Files

#### Model/ApplicationUser.cs
```csharp
using Microsoft.AspNetCore.Identity;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel MazzDataAnnotations.Schema;

namespace EShoppingZone.Models
{
    public class ApplicationUser : IdentityUser<int>
    {
        [Required]
        [StringLength(100)]
        public string FullName { get; set; }

        [Required]
        public int RoleId { get; set; }

        [ForeignKey("RoleId")]
        public Role Role { get; set; }

        public string About { get; set; }
        public DateTime? DateOfBirth { get; set; }
        public string Gender { get; set; }
        public string Image { get; set; }

        public List<Address> Addresses { get; set; } = new List<Address>();
    }
}
```

#### Context/EShoppingZoneDbContext.cs
```csharp
using EShoppingZone.Models;
using EShoppingZone.Profile;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;

namespace EShoppingZone.Context
{
    public class EShoppingZoneDbContext : IdentityDbContext<ApplicationUser, IdentityRole<int>, int>
    {
        public EShoppingZoneDbContext(DbContextOptions<EShoppingZoneDbContext> options)
            : base(options)
        {
        }

        public DbSet<Role> Roles { get; set; }
        public DbSet<Address> Addresses { get; set; }
    }
}
```

#### DTOs/AuthDTOs.cs
```csharp
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.DTOs
{
    public class LoginRequest
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; }

        [Required]
        [StringLength(100, MinimumLength = 6)]
        public string Password { get; set; }
    }

    public class LoginResponse
    {
        public string Token { get; set; }
        public int UserId { get; set; }
        public string Email { get; set; }
        public string RoleName { get; set; }
    }

    public class RegisterRequest
    {
        [Required]
        [StringLength(100)]
        public string FullName { get; set; }

        [Required]
        [EmailAddress]
        public string Email { get; set; }

        [Required]
        [StringLength(100, MinimumLength = 6)]
        public string Password { get; set; }

        [Required]
        [Range(1, int.MaxValue)]
        public int RoleId { get; set; }
    }

    public class PasswordResetRequest
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; }
    }

    public class PasswordResetConfirmRequest
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; }

        [Required]
        public string Token { get; set; }

        [Required]
        [StringLength(100, MinimumLength = 6)]
        public string NewPassword { get; set; }
    }
}
```

#### Interface/IAuthService.cs
```csharp
using EShoppingZone.DTOs;
using System.Threading.Tasks;

namespace EShoppingZone.Interface
{
    public interface IAuthService
    {
        Task<ResponseDTO<LoginResponse>> LoginAsync(LoginRequest request);
        Task<ResponseDTO<bool>> RegisterAsync(RegisterRequest request);
        Task<ResponseDTO<bool>> RequestPasswordResetAsync(PasswordResetRequest request);
        Task<ResponseDTO<bool>> ConfirmPasswordResetAsync(PasswordResetConfirmRequest request);
    }
}
```

#### Service/AuthService.cs
```csharp
using EShoppingZone.Interface;
using EShoppingZone.DTOs;
using EShoppingZone.Models;
using Microsoft.AspNetCore.Identity;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using System.Threading.Tasks;
using System;

namespace EShoppingZone.Services
{
    public class AuthService : IAuthService
    {
        private readonly UserManager<ApplicationUser> _userManager;
        private readonly SignInManager<ApplicationUser> _signInManager;
        private readonly IConfiguration _configuration;
        private readonly IEmailService _emailService;

        public AuthService(UserManager<ApplicationUser> userManager, SignInManager<ApplicationUser> signInManager,
            IConfiguration configuration, IEmailService emailService)
        {
            _userManager = userManager;
            _signInManager = signInManager;
            _configuration = configuration;
            _emailService = emailService;
        }

        public async Task<ResponseDTO<LoginResponse>> LoginAsync(LoginRequest request)
        {
            var user = await _userManager.FindByEmailAsync(request.Email);
            if (user == null)
                return new ResponseDTO<LoginResponse>
                {
                    Success = false,
                    Message = "Invalid email or password.",
                    Data = null
                };

            var result = await _signInManager.CheckPasswordSignInAsync(user, request.Password, false);
            if (!result.Succeeded)
                return new ResponseDTO<LoginResponse>
                {
                    Success = false,
                    Message = "Invalid email or password.",
                    Data = null
                };

            var token = await GenerateJwtToken(user);
            var response = new LoginResponse
            {
                Token = token,
                UserId = user.Id,
                Email = user.Email,
                RoleName = user.Role?.Name
            };

            return new ResponseDTO<LoginResponse>
            {
                Success = true,
                Message = "Login successful.",
                Data = response
            };
        }

        public async Task<ResponseDTO<bool>> RegisterAsync(RegisterRequest request)
        {
            var user = new ApplicationUser
            {
                UserName = request.Email,
                Email = request.Email,
                FullName = request.FullName,
                RoleId = request.RoleId
            };

            var result = await _userManager.CreateAsync(user, request.Password);
            if (!result.Succeeded)
                return new ResponseDTO<bool>
                {
                    Success = false,
                    Message = string.Join("; ", result.Errors.Select(e => e.Description)),
                    Data = false
                };

            return new ResponseDTO<bool>
            {
                Success = true,
                Message = "Registration successful.",
                Data = true
            };
        }

        public async Task<ResponseDTO<bool>> RequestPasswordResetAsync(PasswordResetRequest request)
        {
            var user = await _userManager.FindByEmailAsync(request.Email);
            if (user == null)
                return new ResponseDTO<bool>
                {
                    Success = false,
                    Message = "Email not found.",
                    Data = false
                };

            var token = await _userManager.GeneratePasswordResetTokenAsync(user);
            var resetLink = $"{_configuration["AppUrl"]}/api/Auth/ResetPassword?email={Uri.EscapeDataString(user.Email)}&token={Uri.EscapeDataString(token)}";
            await _emailService.SendEmailAsync(user.Email, "Password Reset", $"Click here to reset your password: {resetLink}");

            return new ResponseDTO<bool>
            {
                Success = true,
                Message = "Password reset email sent.",
                Data = true
            };
        }

        public async Task<ResponseDTO<bool>> ConfirmPasswordResetAsync(PasswordResetConfirmRequest request)
        {
            var user = await _userManager.FindByEmailAsync(request.Email);
            if (user == null)
                return new ResponseDTO<bool>
                {
                    Success = false,
                    Message = "Email not found.",
                    Data = false
                };

            var result = await _userManager.ResetPasswordAsync(user, request.Token, request.NewPassword);
            if (!result.Succeeded)
                return new ResponseDTO<bool>
                {
                    Success = false,
                    Message = string.Join("; ", result.Errors.Select(e => e.Description)),
                    Data = false
                };

            return new ResponseDTO<bool>
            {
                Success = true,
                Message = "Password reset successfully.",
                Data = true
            };
        }

        private async Task<string> GenerateJwtToken(ApplicationUser user)
        {
            var claims = new[]
            {
                new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
                new Claim(JwtRegisteredClaimNames.Email, user.Email),
                new Claim(ClaimTypes.Role, user.Role?.Name ?? "User"),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
            };

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

            var token = new JwtSecurityToken(
                issuer: _configuration["Jwt:Issuer"],
                audience: _configuration["Jwt:Audience"],
                claims: claims,
                expires: DateTime.Now.AddHours(1),
                signingCredentials: creds);

            return new JwtSecurityTokenHandler().WriteToken(token);
        }
    }
}
```

#### Interface/IEmailService.cs
```csharp
using System.Threading.Tasks;

namespace EShoppingZone.Interface
{
    public interface IEmailService
    {
        Task SendEmailAsync(string toEmail, string subject, string body);
    }
}
```

#### Service/EmailService.cs
```csharp
using EShoppingZone.Interface;
using System.Threading.Tasks;
using System.Net.Mail;
using System.Net;

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
            var smtpClient = new SmtpClient(_configuration["Smtp:Host"])
            {
                Port = int.Parse(_configuration["Smtp:Port"]),
                Credentials = new NetworkCredential(_configuration["Smtp:Username"], _configuration["Smtp:Password"]),
                EnableSsl = true,
            };

            var mailMessage = new MailMessage
            {
                From = new MailAddress(_configuration["Smtp:FromEmail"]),
                Subject = subject,
                Body = body,
                IsBodyHtml = true,
            };
            mailMessage.To.Add(toEmail);

            await smtpClient.SendMailAsync(mailMessage);
        }
    }
}
```

#### Controller/AuthController.cs
```csharp
using EShoppingZone.Interface;
using EShoppingZone.DTOs;
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;
using System.Linq;

namespace EShoppingZone.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class AuthController : ControllerBase
    {
        private readonly IAuthService _authService;

        public AuthController(IAuthService authService)
        {
            _authService = authService;
        }

        [HttpPost("login")]
        public async Task<IActionResult> Login([FromBody] LoginRequest request)
        {
            if (!ModelState.IsValid)
            {
                var errors = ModelState.Values
                    .SelectMany(v => v.Errors)
                    .Select(e => e.ErrorMessage)
                    .ToList();
                return BadRequest(new ResponseDTO<LoginResponse>
                {
                    Success = false,
                    Message = "Validation errors: " + string.Join("; ", errors),
                    Data = null
                });
            }

            var result = await _authService.LoginAsync(request);
            if (!result.Success)
                return BadRequest(result);

            return Ok(result);
        }

        [HttpPost("register")]
        public async Task<IActionResult> Register([FromBody] RegisterRequest request)
        {
            if (!ModelState.IsValid)
            {
                var errors = ModelState.Values
                    .SelectMany(v => v.Errors)
                    .Select(e => e.ErrorMessage)
                    .ToList();
                return BadRequest(new ResponseDTO<bool>
                {
                    Success = false,
                    Message = "Validation errors: " + string.Join("; ", errors),
                    Data = false
                });
            }

            var result = await _authService.RegisterAsync(request);
            if (!result.Success)
                return BadRequest(result);

            return Ok(result);
        }

        [HttpPost("request-password-reset")]
        public async Task<IActionResult> RequestPasswordReset([FromBody] PasswordResetRequest request)
        {
            if (!ModelState.IsValid)
            {
                var errors = ModelState.Values
                    .SelectMany(v => v.Errors)
                    .Select(e => e.ErrorMessage)
                    .ToList();
                return BadRequest(new ResponseDTO<bool>
                {
                    Success = false,
                    Message = "Validation errors: " + string.Join("; ", errors),
                    Data = false
                });
            }

            var result = await _authService.RequestPasswordResetAsync(request);
            if (!result.Success)
                return BadRequest(result);

            return Ok(result);
        }

        [HttpPost("reset-password")]
        public async Task<IActionResult> ConfirmPasswordReset([FromBody] PasswordResetConfirmRequest request)
        {
            if (!ModelState.IsValid)
            {
                var errors = ModelState.Values
                    .SelectMany(v => v.Errors)
                    .Select(e => e.ErrorMessage)
                    .ToList();
                return BadRequest(new ResponseDTO<bool>
                {
                    Success = false,
                    Message = "Validation errors: " + string.Join("; ", errors),
                    Data = false
                });
            }

            var result = await _authService.ConfirmPasswordResetAsync(request);
            if (!result.Success)
                return BadRequest(result);

            return Ok(result);
        }
    }
}
```

#### Program.cs
```csharp
using EShoppingZone.Context;
using EShoppingZone.Interface;
using EShoppingZone.Models;
using EShoppingZone.Repository;
using EShoppingZone.Services;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Add services
builder.Services.AddControllers().AddNewtonsoftJson();
builder.Services.AddDbContext<EShoppingZoneDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
builder.Services.AddIdentity<ApplicationUser, IdentityRole<int>>()
    .AddEntityFrameworkStores<EShoppingZoneDbContext>()
    .AddDefaultTokenProviders();
builder.Services.AddScoped<IProfileRepository, ProfileRepository>();
builder.Services.AddScoped<IProfileService, ProfileService>();
builder.Services.AddScoped<IAuthService, AuthService>();
builder.Services.AddScoped<IEmailService, EmailService>();

// JWT Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
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

var app = builder.Build();

// Configure middleware
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

#### appsettings.json
```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=EShoppingZone;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "Jwt": {
    "Key": "YourSecretKeyHere1234567890abcdef",
    "Issuer": "EShoppingZone",
    "Audience": "EShoppingZone"
  },
  "Smtp": {
    "Host": "smtp.gmail.com",
    "Port": "587",
    "Username": "your-email@gmail.com",
    "Password": "your-app-specific-password",
    "FromEmail": "your-email@gmail.com"
  },
  "AppUrl": "https://localhost:5001"
}
```

### Update Profile Module for Identity
Since `UserProfile` is replaced by `ApplicationUser`, update the Profile module files to use `ApplicationUser`.

#### Interface/IProfileService.cs
```csharp
using EShoppingZone.DTOs;
using System.Threading.Tasks;
using System.Collections.Generic;

namespace EShoppingZone.Interface
{
    public interface IProfileService
    {
        Task<ResponseDTO<ProfileResponse>> GetProfileAsync(int userId);
        Task<ResponseDTO<ProfileResponse>> UpdateProfileAsync(int userId, UpdateProfileRequest request);
        Task<ResponseDTO<bool>> DeleteProfileAsync(int userId);
        Task<ResponseDTO<AddressResponse>> CreateAddressAsync(int userId, CreateAddressRequest request);
        Task<ResponseDTO<AddressResponse>> UpdateAddressAsync(int userId, int addressId, UpdateAddressRequest request);
        Task<ResponseDTO<bool>> DeleteAddressAsync(int userId, int addressId);
        Task<ResponseDTO<List<AddressResponse>>> GetAddressesByProfileIdAsync(int userId);
        Task<ResponseDTO<AddressResponse>> GetAddressByIdAsync(int userId, int addressId);
        Task<ResponseDTO<List<RoleResponse>>> GetRolesAsync();
    }
}
```

#### Service/ProfileService.cs
```csharp
using EShoppingZone.Interface;
using EShoppingZone.DTOs;
using EShoppingZone.Models;
using System.Threading.Tasks;
using System.Collections.Generic;
using System.Linq;
using Microsoft.EntityFrameworkCore;

namespace EShoppingZone.Services
{
    public class ProfileService : IProfileService
    {
        private readonly IProfileRepository _repository;

        public ProfileService(IProfileRepository repository)
        {
            _repository = repository;
        }

        public async Task<ResponseDTO<ProfileResponse>> GetProfileAsync(int userId)
        {
            var user = await _repository.GetUserAsync(userId);
            if (user == null)
                return new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = "User not found.",
                    Data = null
                };

            var response = new ProfileResponse
            {
                UserId = user.Id,
                FullName = user.FullName,
                Email = user.Email,
                MobileNumber = user.PhoneNumber,
                About = user.About,
                DateOfBirth = user.DateOfBirth,
                Gender = user.Gender,
                Image = user.Image,
                RoleId = user.RoleId,
                RoleName = user.Role?.Name,
                Addresses = user.Addresses?.Select(a => new AddressResponse
                {
                    Id = a.Id,
                    HouseNumber = a.HouseNumber,
                    StreetName = a.StreetName,
                    ColonyName = a.ColonyName,
                    City = a.City,
                    State = a.State,
                    Pincode = a.Pincode
                }).ToList() ?? new List<AddressResponse>()
            };

            return new ResponseDTO<ProfileResponse>
            {
                Success = true,
                Message = "Profile retrieved successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<ProfileResponse>> UpdateProfileAsync(int userId, UpdateProfileRequest request)
        {
            var user = await _repository.GetUserAsync(userId);
            if (user == null)
                return new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = "User not found.",
                    Data = null
                };

            user.FullName = request.FullName ?? user.FullName;
            user.Email = request.Email ?? user.Email;
            user.PhoneNumber = request.MobileNumber != 0 ? request.MobileNumber.ToString() : user.PhoneNumber;
            user.About = request.About ?? user.About;
            user.DateOfBirth = request.DateOfBirth ?? user.DateOfBirth;
            user.Gender = request.Gender ?? user.Gender;
            user.Image = request.Image ?? user.Image;

            await _repository.UpdateUserAsync(user);

            var response = new ProfileResponse
            {
                UserId = user.Id,
                FullName = user.FullName,
                Email = user.Email,
                MobileNumber = user.PhoneNumber,
                About = user.About,
                DateOfBirth = user.DateOfBirth,
                Gender = user.Gender,
                Image = user.Image,
                RoleId = user.RoleId,
                RoleName = user.Role?.Name,
                Addresses = user.Addresses?.Select(a => new AddressResponse
                {
                    Id = a.Id,
                    HouseNumber = a.HouseNumber,
                    StreetName = a.StreetName,
                    ColonyName = a.ColonyName,
                    City = a.City,
                    State = a.State,
                    Pincode = a.Pincode
                }).ToList() ?? new List<AddressResponse>()
            };

            return new ResponseDTO<ProfileResponse>
            {
                Success = true,
                Message = "Profile updated successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<bool>> DeleteProfileAsync(int userId)
        {
            var user = await _repository.GetUserAsync(userId);
            if (user == null)
                return new ResponseDTO<bool>
                {
                    Success = false,
                    Message = "User not found.",
                    Data = false
                };

            var hasOrders = await _repository.HasOrdersAsync(userId);
            if (hasOrders)
                return new ResponseDTO<bool>
                {
                    Success = false,
                    Message = "Cannot delete profile with existing orders.",
                    Data = false
                };

            await _repository.DeleteUserAsync(user);

            return new ResponseDTO<bool>
            {
                Success = true,
                Message = "Profile deleted successfully.",
                Data = true
            };
        }

        public async Task<ResponseDTO<AddressResponse>> CreateAddressAsync(int userId, CreateAddressRequest request)
        {
            var user = await _repository.GetUserAsync(userId);
            if (user == null)
                return new ResponseDTO<AddressResponse>
                {
                    Success = false,
                    Message = "User not found.",
                    Data = null
                };

            var address = new Address
            {
                HouseNumber = request.HouseNumber,
                StreetName = request.StreetName,
                ColonyName = request.ColonyName,
                City = request.City,
                State = request.State,
                Pincode = request.Pincode,
                ProfileId = userId
            };

            await _repository.AddAddressAsync(address);

            var response = new AddressResponse
            {
                Id = address.Id,
                HouseNumber = address.HouseNumber,
                StreetName = address.StreetName,
                ColonyName = address.ColonyName,
                City = address.City,
                State = address.State,
                Pincode = address.Pincode
            };

            return new ResponseDTO<AddressResponse>
            {
                Success = true,
                Message = "Address created successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<AddressResponse>> UpdateAddressAsync(int userId, int addressId, UpdateAddressRequest request)
        {
            var address = await _repository.GetAddressAsync(addressId);
            if (address == null || address.ProfileId != userId)
                return new ResponseDTO<AddressResponse>
                {
                    Success = false,
                    Message = "Address not found or does not belong to user.",
                    Data = null
                };

            address.HouseNumber = request.HouseNumber;
            address.StreetName = request.StreetName;
            address.ColonyName = request.ColonyName;
            address.City = request.City;
            address.State = request.State;
            address.Pincode = request.Pincode;

            await _repository.UpdateAddressAsync(address);

            var response = new AddressResponse
            {
                Id = address.Id,
                HouseNumber = address.HouseNumber,
                StreetName = address.StreetName,
                ColonyName = address.ColonyName,
                City = address.City,
                State = address.State,
                Pincode = address.Pincode
            };

            return new ResponseDTO<AddressResponse>
            {
                Success = true,
                Message = "Address updated successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<bool>> DeleteAddressAsync(int userId, int addressId)
        {
            var address = await _repository.GetAddressAsync(addressId);
            if (address == null || address.ProfileId != userId)
                return new ResponseDTO<bool>
                {
                    Success = false,
                    Message = "Address not found or does not belong to user.",
                    Data = false
                };

            await _repository.DeleteAddressAsync(address);

            return new ResponseDTO<bool>
            {
                Success = true,
                Message = "Address deleted successfully.",
                Data = true
            };
        }

        public async Task<ResponseDTO<List<AddressResponse>>> GetAddressesByProfileIdAsync(int userId)
        {
            var user = await _repository.GetUserAsync(userId);
            if (user == null)
                return new ResponseDTO<List<AddressResponse>>
                {
                    Success = false,
                    Message = "User not found.",
                    Data = null
                };

            var addresses = await _repository.GetAddressesByProfileIdAsync(userId);
            var response = addresses.Select(a => new AddressResponse
            {
                Id = a.Id,
                HouseNumber = a.HouseNumber,
                StreetName = a.StreetName,
                ColonyName = a.ColonyName,
                City = a.City,
                State = a.State,
                Pincode = a.Pincode
            }).ToList();

            return new ResponseDTO<List<AddressResponse>>
            {
                Success = true,
                Message = "Addresses retrieved successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<AddressResponse>> GetAddressByIdAsync(int userId, int addressId)
        {
            var address = await _repository.GetAddressAsync(addressId);
            if (address == null || address.ProfileId != userId)
                return new ResponseDTO<AddressResponse>
                {
                    Success = false,
                    Message = "Address not found or does not belong to user.",
                    Data = null
                };

            var response = new AddressResponse
            {
                Id = address.Id,
                HouseNumber = address.HouseNumber,
                StreetName = address.StreetName,
                ColonyName = address.ColonyName,
                City = address.City,
                State = address.State,
                Pincode = address.Pincode
            };

            return new ResponseDTO<AddressResponse>
            {
                Success = true,
                Message = "Address retrieved successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<List<RoleResponse>>> GetRolesAsync()
        {
            var roles = await _repository.GetRolesAsync();
            var response = roles.Select(r => new RoleResponse
            {
                RoleId = r.RoleId,
                Name = r.Name
            }).ToList();

            return new ResponseDTO<List<RoleResponse>>
            {
                Success = true,
                Message = "Roles retrieved successfully.",
                Data = response
            };
        }
    }
}
```

#### Interface/IProfileRepository.cs
```csharp
using EShoppingZone.Models;
using EShoppingZone.Profile;
using System.Threading.Tasks;
using System.Collections.Generic;

namespace EShoppingZone.Interface
{
    public interface IProfileRepository
    {
        Task<ApplicationUser> GetUserAsync(int userId);
        Task UpdateUserAsync(ApplicationUser user);
        Task DeleteUserAsync(ApplicationUser user);
        Task AddAddressAsync(Address address);
        Task<Address> GetAddressAsync(int addressId);
        Task UpdateAddressAsync(Address address);
        Task DeleteAddressAsync(Address address);
        Task<List<Address>> GetAddressesByProfileIdAsync(int userId);
        Task<bool> HasOrdersAsync(int userId);
        Task<Role> GetRoleAsync(int roleId);
        Task<List<Role>> GetRolesAsync();
    }
}
```

#### Repository/ProfileRepository.cs
```csharp
using EShoppingZone.Context;
using EShoppingZone.Models;
using EShoppingZone.Profile;
using Microsoft.EntityFrameworkCore;
using System.Threading.Tasks;
using System.Collections.Generic;

namespace EShoppingZone.Repository
{
    public class ProfileRepository : IProfileRepository
    {
        private readonly EShoppingZoneDbContext _context;

        public ProfileRepository(EShoppingZoneDbContext context)
        {
            _context = context;
        }

        public async Task<ApplicationUser> GetUserAsync(int userId)
        {
            return await _context.Users
                .Include(u => u.Role)
                .Include(u => u.Addresses)
                .FirstOrDefaultAsync(u => u.Id == userId);
        }

        public async Task UpdateUserAsync(ApplicationUser user)
        {
            _context.Users.Update(user);
            await _context.SaveChangesAsync();
        }

        public async Task DeleteUserAsync(ApplicationUser user)
        {
            _context.Users.Remove(user);
            await _context.SaveChangesAsync();
        }

        public async Task AddAddressAsync(Address address)
        {
            await _context.Addresses.AddAsync(address);
            await _context.SaveChangesAsync();
        }

        public async Task<Address> GetAddressAsync(int addressId)
        {
            return await _context.Addresses
                .FirstOrDefaultAsync(a => a.Id == addressId);
        }

        public async Task UpdateAddressAsync(Address address)
        {
            _context.Addresses.Update(address);
            await _context.SaveChangesAsync();
        }

        public async Task DeleteAddressAsync(Address address)
        {
            _context.Addresses.Remove(address);
            await _context.SaveChangesAsync();
        }

        public async Task<List<Address>> GetAddressesByProfileIdAsync(int userId)
        {
            return await _context.Addresses
                .Where(a => a.ProfileId == userId)
                .ToListAsync();
        }

        public async Task<bool> HasOrdersAsync(int userId)
        {
            return await _context.Orders
                .AnyAsync(o => o.ProfileId == userId);
        }

        public async Task<Role> GetRoleAsync(int roleId)
        {
            return await _context.Roles
                .FirstOrDefaultAsync(r => r.RoleId == roleId);
        }

        public async Task<List<Role>> GetRolesAsync()
        {
            return await _context.Roles.ToListAsync();
        }
    }
}
```

#### Controller/ProfileController.cs
```csharp
using EShoppingZone.Interface;
using EShoppingZone.DTOs;
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;
using System.Collections.Generic;
using System.Linq;
using Microsoft.AspNetCore.Authorization;
using System.Security.Claims;

namespace EShoppingZone.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class ProfileController : ControllerBase
    {
        private readonly IProfileService _service;

        public ProfileController(IProfileService service)
        {
            _service = service;
        }

        [Authorize]
        [HttpGet("{userId}")]
        public async Task<IActionResult> GetProfile(int userId)
        {
            var jwtUserId = int.Parse(User.FindFirst("sub")?.Value ?? "0");
            if (jwtUserId != userId)
                return Unauthorized(new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = "Unauthorized access to profile.",
                    Data = null
                });

            var result = await _service.GetProfileAsync(userId);
            if (!result.Success)
                return NotFound(result);

            return Ok(result);
        }

        [Authorize]
        [HttpPut("{userId}")]
        public async Task<IActionResult> UpdateProfile(int userId, [FromBody] UpdateProfileRequest request)
        {
            if (!ModelState.IsValid)
            {
                var errors = ModelState.Values
                    .SelectMany(v => v.Errors)
                    .Select(e => e.ErrorMessage)
                    .ToList();
                return BadRequest(new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = "Validation errors: " + string.Join("; ", errors),
                    Data = null
                });
            }

            var jwtUserId = int.Parse(User.FindFirst("sub")?.Value ?? "0");
            if (jwtUserId != userId)
                return Unauthorized(new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = "Unauthorized access to profile.",
                    Data = null
                });

            var result = await _service.UpdateProfileAsync(userId, request);
            if (!result.Success)
                return NotFound(result);

            return Ok(result);
        }

        [Authorize]
        [HttpDelete("{userId}")]
        public async Task<IActionResult> DeleteProfile(int userId)
        {
            var jwtUserId = int.Parse(User.FindFirst("sub")?.Value ?? "0");
            if (jwtUserId != userId)
                return Unauthorized(new ResponseDTO<bool>
                {
                    Success = false,
                    Message = "Unauthorized access to profile.",
                    Data = false
                });

            var result = await _service.DeleteProfileAsync(userId);
            if (!result.Success)
                return NotFound(result);

            return Ok(result);
        }

        [Authorize]
        [HttpPost("addresses")]
        public async Task<IActionResult> CreateAddress([FromBody] CreateAddressRequest request)
        {
            if (!ModelState.IsValid)
            {
                var errors = ModelState.Values
                    .SelectMany(v => v.Errors)
                    .Select(e => e.ErrorMessage)
                    .ToList();
                return BadRequest(new ResponseDTO<AddressResponse>
                {
                    Success = false,
                    Message = "Validation errors: " + string.Join("; ", errors),
                    Data = null
                });
            }

            var userId = int.Parse(User.FindFirst("sub")?.Value ?? "0");
            var result = await _service.CreateAddressAsync(userId, request);
            if (!result.Success)
                return NotFound(result);

            return CreatedAtAction(nameof(GetAddressById), new { addressId = result.Data.Id }, result);
        }

        [Authorize]
        [HttpPut("addresses/{addressId}")]
        public async Task<IActionResult> UpdateAddress(int addressId, [FromBody] UpdateAddressRequest request)
        {
            if (!ModelState.IsValid)
            {
                var errors = ModelState.Values
                    .SelectMany(v => v.Errors)
                    .Select(e => e.ErrorMessage)
                    .ToList();
                return BadRequest(new ResponseDTO<AddressResponse>
                {
                    Success = false,
                    Message = "Validation errors: " + string.Join("; ", errors),
                    Data = null
                });
            }

            var userId = int.Parse(User.FindFirst("sub")?.Value ?? "0");
            var result = await _service.UpdateAddressAsync(userId, addressId, request);
            if (!result.Success)
                return NotFound(result);

            return Ok(result);
        }

        [Authorize]
        [HttpDelete("addresses/{addressId}")]
        public async Task<IActionResult> DeleteAddress(int addressId)
        {
            var userId = int.Parse(User.FindFirst("sub")?.Value ?? "0");
            var result = await _service.DeleteAddressAsync(userId, addressId);
            if (!result.Success)
                return NotFound(result);

            return Ok(result);
        }

        [Authorize]
        [HttpGet("addresses")]
        public async Task<IActionResult> GetAddressesByProfileId()
        {
            var userId = int.Parse(User.FindFirst("sub")?.Value ?? "0");
            var result = await _service.GetAddressesByProfileIdAsync(userId);
            if (!result.Success)
                return NotFound(result);

            return Ok(result);
        }

        [Authorize]
        [HttpGet("addresses/{addressId}")]
        public async Task<IActionResult> GetAddressById(int addressId)
        {
            var userId = int.Parse(User.FindFirst("sub")?.Value ?? "0");
            var result = await _service.GetAddressByIdAsync(userId, addressId);
            if (!result.Success)
                return NotFound(result);

            return Ok(result);
        }

        [HttpGet("roles")]
        public async Task<IActionResult> GetRoles()
        {
            var result = await _service.GetRolesAsync();
            return Ok(result);
        }
    }
}
```

#### DTOs/ProfileDTOs.cs (Updated)
```csharp
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.DTOs
{
    public class ProfileResponse
    {
        public int UserId { get; set; }
        public string FullName { get; set; }
        public string Email { get; set; }
        public string MobileNumber { get; set; }
        public string About { get; set; }
        public DateTime? DateOfBirth { get; set; }
        public string Gender { get; set; }
        public string Image { get; set; }
        public int RoleId { get; set; }
        public string RoleName { get; set; }
        public List<AddressResponse> Addresses { get; set; }
    }

    public class UpdateProfileRequest
    {
        [StringLength(100)]
        public string FullName { get; set; }

        [EmailAddress]
        public string Email { get; set; }

        public long MobileNumber { get; set; }

        public string About { get; set; }
        public DateTime? DateOfBirth { get; set; }
        public string Gender { get; set; }
        public string Image { get; set; }
    }
}
```

### Migration Steps
1. **Remove Old Tables** (if `UserProfile` exists):
   ```bash
   dotnet ef migrations add RemoveUserProfile
   ```
   In migration file:
   ```csharp
   public partial class RemoveUserProfile : Migration
   {
       protected override void Up(MigrationBuilder migrationBuilder)
       {
           migrationBuilder.DropTable(name: "Profiles");
       }

       protected override void Down(MigrationBuilder migrationBuilder)
       {
           // Recreate Profiles table if needed
       }
   }
   ```
2. **Add Identity Tables**:
   ```bash
   dotnet ef migrations add AddIdentityTables
   dotnet ef database update
   ```

### Testing
1. **Run**: `dotnet run`
2. **Swagger**: `https://localhost:5001/swagger`
3. **Register**:
   ```http
   POST /api/Auth/register
   Content-Type: application/json
   {
       "fullName": "John Doe",
       "email": "john@example.com",
       "password": "Password123!",
       "roleId": 1
   }
   ```
   **Expected**:
   ```json
   {
       "success": true,
       "message": "Registration successful.",
       "data": true
   }
   ```
4. **Login**:
   ```http
   POST /api/Auth/login
   Content-Type: application/json
   {
       "email": "john@example.com",
       "password": "Password123!"
   }
   ```
   **Expected**:
   ```json
   {
       "success": true,
       "message": "Login successful.",
       "data": {
           "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
           "userId": 1,
           "email": "john@example.com",
           "roleName": "Customer"
       }
   }
   ```
5. **Request Password Reset**:
   ```http
   POST /api/Auth/request-password-reset
   Content-Type: application/json
   {
       "email": "john@example.com"
   }
   ```
   **Expected**:
   ```json
   {
       "success": true,
       "message": "Password reset email sent.",
       "data": true
   }
   ```
   **Verify**: Check email for reset link.
6. **Confirm Password Reset**:
   ```http
   POST /api/Auth/reset-password
   Content-Type: application/json
   {
       "email": "john@example.com",
       "token": "<token-from-email>",
       "newPassword": "NewPassword123!"
   }
   ```
   **Expected**:
   ```json
   {
       "success": true,
       "message": "Password reset successfully.",
       "data": true
   }
   ```
7. **Test Profile Endpoint**:
   ```http
   GET /api/Profile/1/addresses
   Authorization: Bearer <token>
   ```
   **Expected**:
   ```json
   {
       "success": true,
       "message": "Addresses retrieved successfully.",
       "data": []
   }
   ```

### Notes
- **Identity**: Replaces `UserProfile` with `ApplicationUser`, using `IdentityUser<int>` for integer keys.
- **JWT**: `sub` claim holds `Id` (maps to `ProfileId`), used in `ProfileController.cs` (April 23, 2025, 14:45).
- **Email**: Configure SMTP (e.g., Gmail) in `appsettings.json`.
- **Profile Module**: Updated to use `ApplicationUser`, `UserId` instead of `ProfileId`.
- **Roles**: `RoleId` in `ApplicationUser` links to custom `Role` table (not Identity roles).

### Next Steps
- Confirm if auth and updated Profile module files work.
- Need tweaks (e.g., role-based authorization, different SMTP)?
- Return to Product module (April 23, 2025, 15:37)?
- Other features (e.g., Cart, Order)?

Let me know what’s next!
