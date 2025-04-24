I'll provide a fresh implementation of the **EShoppingZone** project focusing solely on the **Profile** module, incorporating your requirements:
- Use the `UserProfile` model during registration.
- Enable CRUD operations for addresses and profile updates after JWT authorization.
- Extract `ProfileId` from the JWT token's claims to avoid manual input in API calls.
- Restrict CRUD operations to the logged-in user's `ProfileId`.

### Project Setup
- **Tech Stack**: .NET 8, ASP.NET Core Web API, EF Core, SQL Server, ASP.NET Core Identity, JWT.
- **Architecture**: Layered (Models, DTOs, Data, Repositories, Services, Controllers).
- **Features**:
  - Register/Login with `UserProfile` (inherits `IdentityUser<int>`).
  - Address CRUD operations (tied to the logged-in user’s `ProfileId`).
  - Profile update (e.g., `FullName`, `About`) for the logged-in user.
  - `ProfileId` extracted from JWT claims.
- **Exclusions**: Product module and other modules ignored as requested.

### Assumptions
- Using `Microsoft.AspNetCore.Identity.EntityFrameworkCore` 8.0.8.
- SQL Server as the database.
- JWT configuration in `appsettings.json`.
- Basic error handling and validation.
- No additional models (e.g., `Product`, `Cart`) or modules.

### Deliverables
- Full code for models, DTOs, DbContext, repositories, services, controllers, and configuration.
- Migration script for the database schema.
- Sample `appsettings.json` and `Program.cs`.
- Testing steps for API endpoints.
- Suggestions for improvements.

### Code Implementation
Below are the complete files, each wrapped in `<xaiArtifact/>` tags with unique UUIDs, proper titles, and content types.

#### 1. Models (Models/UserProfile.cs, Role.cs, Address.cs)
The `UserProfile` inherits `IdentityUser<int>`, and `Address` is tied to `UserProfile`.

```x-csharp
using Microsoft.AspNetCore.Identity;
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace EShoppingZone.Models
{
    public class UserProfile : IdentityUser<int>
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

```x-csharp
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.Models
{
    public class Role
    {
        public int RoleId { get; set; }

        [Required]
        [StringLength(50)]
        public string Name { get; set; }
    }
}
```

```x-csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace EShoppingZone.Models
{
    public class Address
    {
        public int Id { get; set; }

        [Required]
        public string HouseNumber { get; set; }

        [Required]
        public string StreetName { get; set; }

        [Required]
        public string ColonyName { get; set; }

        [Required]
        public string City { get; set; }

        [Required]
        public string State { get; set; }

        [Required]
        public string Pincode { get; set; }

        public int ProfileId { get; set; }

        [ForeignKey("ProfileId")]
        public UserProfile Profile { get; set; }
    }
}
```

#### 2. DTOs (DTOs/AuthDtos.cs, ProfileDtos.cs, AddressDtos.cs)
DTOs for registration, login, profile updates, and address CRUD.

```x-csharp
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.DTOs
{
    public class RegisterRequest
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; }

        [Required]
        [StringLength(100)]
        public string FullName { get; set; }

        [Required]
        public int RoleId { get; set; }

        [Required]
        [StringLength(100, MinimumLength = 6)]
        public string Password { get; set; }

        public string About { get; set; }
        public DateTime? DateOfBirth { get; set; }
        public string Gender { get; set; }
        public string Image { get; set; }
    }

    public class LoginRequest
    {
        [Required]
        [EmailAddress]
        public string Email { get; set; }

        [Required]
        public string Password { get; set; }
    }

    public class LoginResponse
    {
        public string Token { get; set; }
        public string FullName { get; set; }
        public int Id { get; set; }
    }
}
```

```x-csharp
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.DTOs
{
    public class ProfileResponse
    {
        public int Id { get; set; }
        public string Email { get; set; }
        public string FullName { get; set; }
        public int RoleId { get; set; }
        public string RoleName { get; set; }
        public string About { get; set; }
        public DateTime? DateOfBirth { get; set; }
        public string Gender { get; set; }
        public string Image { get; set; }
    }

    public class UpdateProfileRequest
    {
        [Required]
        [StringLength(100)]
        public string FullName { get; set; }

        public string About { get; set; }
        public DateTime? DateOfBirth { get; set; }
        public string Gender { get; set; }
        public string Image { get; set; }
    }
}
```

```x-csharp
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.DTOs
{
    public class AddressDto
    {
        public int Id { get; set; }
        public string HouseNumber { get; set; }
        public string StreetName { get; set; }
        public string ColonyName { get; set; }
        public string City { get; set; }
        public string State { get; set; }
        public string Pincode { get; set; }
    }

    public class CreateAddressRequest
    {
        [Required]
        public string HouseNumber { get; set; }
        [Required]
        public string StreetName { get; set; }
        [Required]
        public string ColonyName { get; set; }
        [Required]
        public string City { get; set; }
        [Required]
        public string State { get; set; }
        [Required]
        public string Pincode { get; set; }
    }

    public class UpdateAddressRequest
    {
        [Required]
        public string HouseNumber { get; set; }
        [Required]
        public string StreetName { get; set; }
        [Required]
        public string ColonyName { get; set; }
        [Required]
        public string City { get; set; }
        [Required]
        public string State { get; set; }
        [Required]
        public string Pincode { get; set; }
    }
}
```

#### 3. DbContext (Data/EShoppingZoneDbContext.cs)
Configures the database schema and relationships.

```x-csharp
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using EShoppingZone.Models;

namespace EShoppingZone.Data
{
    public class EShoppingZoneDbContext : IdentityDbContext<UserProfile, IdentityRole<int>, int>
    {
        public EShoppingZoneDbContext(DbContextOptions<EShoppingZoneDbContext> options)
            : base(options)
        {
        }

        public DbSet<UserProfile> Profiles { get; set; }
        public DbSet<Role> Roles { get; set; }
        public DbSet<Address> Addresses { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            base.OnModelCreating(modelBuilder);

            // Rename AspNetUsers to Profiles
            modelBuilder.Entity<UserProfile>().ToTable("Profiles");

            // UserProfile configuration
            modelBuilder.Entity<UserProfile>()
                .HasKey(up => up.Id);

            modelBuilder.Entity<UserProfile>()
                .HasOne(up => up.Role)
                .WithMany()
                .HasForeignKey(up => up.RoleId)
                .OnDelete(DeleteBehavior.Restrict);

            modelBuilder.Entity<UserProfile>()
                .HasMany(up => up.Addresses)
                .WithOne(a => a.Profile)
                .HasForeignKey(a => a.ProfileId)
                .OnDelete(DeleteBehavior.Cascade);

            // Role configuration
            modelBuilder.Entity<Role>()
                .HasKey(r => r.RoleId);

            modelBuilder.Entity<Role>()
                .HasIndex(r => r.Name)
                .IsUnique();

            // Address configuration
            modelBuilder.Entity<Address>()
                .HasKey(a => a.Id);
        }
    }
}
```

#### 4. Repositories (Repositories/ProfileRepository.cs, AddressRepository.cs)
Repositories for profile and address operations.

```x-csharp
using Microsoft.EntityFrameworkCore;
using System.Threading.Tasks;
using EShoppingZone.Data;
using EShoppingZone.Models;

namespace EShoppingZone.Repositories
{
    public class ProfileRepository : IProfileRepository
    {
        private readonly EShoppingZoneDbContext _context;

        public ProfileRepository(EShoppingZoneDbContext context)
        {
            _context = context;
        }

        public async Task<UserProfile> GetProfileByIdAsync(int id)
        {
            return await _context.Profiles
                .Include(up => up.Role)
                .FirstOrDefaultAsync(up => up.Id == id);
        }

        public async Task UpdateProfileAsync(UserProfile profile)
        {
            _context.Profiles.Update(profile);
            await _context.SaveChangesAsync();
        }
    }

    public interface IProfileRepository
    {
        Task<UserProfile> GetProfileByIdAsync(int id);
        Task UpdateProfileAsync(UserProfile profile);
    }
}
```

```x-csharp
using Microsoft.EntityFrameworkCore;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using EShoppingZone.Data;
using EShoppingZone.Models;

namespace EShoppingZone.Repositories
{
    public class AddressRepository : IAddressRepository
    {
        private readonly EShoppingZoneDbContext _context;

        public AddressRepository(EShoppingZoneDbContext context)
        {
            _context = context;
        }

        public async Task<List<Address>> GetAddressesByProfileIdAsync(int profileId)
        {
            return await _context.Addresses
                .Where(a => a.ProfileId == profileId)
                .ToListAsync();
        }

        public async Task<Address> GetAddressByIdAsync(int id, int profileId)
        {
            return await _context.Addresses
                .FirstOrDefaultAsync(a => a.Id == id && a.ProfileId == profileId);
        }

        public async Task<Address> CreateAddressAsync(Address address)
        {
            _context.Addresses.Add(address);
            await _context.SaveChangesAsync();
            return address;
        }

        public async Task<Address> UpdateAddressAsync(Address address)
        {
            _context.Addresses.Update(address);
            await _context.SaveChangesAsync();
            return address;
        }

        public async Task<bool> DeleteAddressAsync(int id, int profileId)
        {
            var address = await _context.Addresses
                .FirstOrDefaultAsync(a => a.Id == id && a.ProfileId == profileId);
            if (address == null)
                return false;

            _context.Addresses.Remove(address);
            await _context.SaveChangesAsync();
            return true;
        }
    }

    public interface IAddressRepository
    {
        Task<List<Address>> GetAddressesByProfileIdAsync(int profileId);
        Task<Address> GetAddressByIdAsync(int id, int profileId);
        Task<Address> CreateAddressAsync(Address address);
        Task<Address> UpdateAddressAsync(Address address);
        Task<bool> DeleteAddressAsync(int id, int profileId);
    }
}
```

#### 5. Services (Services/AuthService.cs, ProfileService.cs, AddressService.cs)
Services for authentication, profile updates, and address CRUD.

```x-csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.Extensions.Configuration;
using Microsoft.IdentityModel.Tokens;
using System;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using EShoppingZone.Models;

namespace EShoppingZone.Services
{
    public class AuthService : IAuthService
    {
        private readonly UserManager<UserProfile> _userManager;
        private readonly IConfiguration _configuration;

        public AuthService(UserManager<UserProfile> userManager, IConfiguration configuration)
        {
            _userManager = userManager;
            _configuration = configuration;
        }

        public async Task<LoginResponse> RegisterAsync(RegisterRequest request)
        {
            var user = new UserProfile
            {
                UserName = request.Email,
                Email = request.Email,
                FullName = request.FullName,
                RoleId = request.RoleId,
                About = request.About,
                DateOfBirth = request.DateOfBirth,
                Gender = request.Gender,
                Image = request.Image
            };

            var result = await _userManager.CreateAsync(user, request.Password);
            if (!result.Succeeded)
            {
                throw new Exception(string.Join(", ", result.Errors.Select(e => e.Description)));
            }

            var token = GenerateJwtToken(user);
            return new LoginResponse
            {
                Token = token,
                FullName = user.FullName,
                Id = user.Id
            };
        }

        public async Task<LoginResponse> LoginAsync(LoginRequest request)
        {
            var user = await _userManager.FindByEmailAsync(request.Email);
            if (user == null || !await _userManager.CheckPasswordAsync(user, request.Password))
            {
                throw new Exception("Invalid email or password.");
            }

            var token = GenerateJwtToken(user);
            return new LoginResponse
            {
                Token = token,
                FullName = user.FullName,
                Id = user.Id
            };
        }

        private string GenerateJwtToken(UserProfile user)
        {
            var claims = new[]
            {
                new Claim(JwtRegisteredClaimNames.Sub, user.Id.ToString()),
                new Claim(JwtRegisteredClaimNames.Email, user.Email),
                new Claim("FullName", user.FullName),
                new Claim("RoleId", user.RoleId.ToString()),
                new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()) // ProfileId for CRUD
            };

            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_configuration["Jwt:Key"]));
            var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

            var token = new JwtSecurityToken(
                issuer: _configuration["Jwt:Issuer"],
                audience: _configuration["Jwt:Audience"],
                claims: claims,
                expires: DateTime.Now.AddDays(1),
                signingCredentials: creds);

            return new JwtSecurityTokenHandler().WriteToken(token);
        }
    }

    public interface IAuthService
    {
        Task<LoginResponse> RegisterAsync(RegisterRequest request);
        Task<LoginResponse> LoginAsync(LoginRequest request);
    }
}
```

```x-csharp
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using EShoppingZone.Models;
using EShoppingZone.Repositories;

namespace EShoppingZone.Services
{
    public class ProfileService : IProfileService
    {
        private readonly IProfileRepository _profileRepository;

        public ProfileService(IProfileRepository profileRepository)
        {
            _profileRepository = profileRepository;
        }

        public async Task<ProfileResponse> GetProfileAsync(int profileId)
        {
            var profile = await _profileRepository.GetProfileByIdAsync(profileId);
            if (profile == null)
                throw new Exception("Profile not found.");

            return new ProfileResponse
            {
                Id = profile.Id,
                Email = profile.Email,
                FullName = profile.FullName,
                RoleId = profile.RoleId,
                RoleName = profile.Role?.Name,
                About = profile.About,
                DateOfBirth = profile.DateOfBirth,
                Gender = profile.Gender,
                Image = profile.Image
            };
        }

        public async Task<ProfileResponse> UpdateProfileAsync(int profileId, UpdateProfileRequest request)
        {
            var profile = await _profileRepository.GetProfileByIdAsync(profileId);
            if (profile == null)
                throw new Exception("Profile not found.");

            profile.FullName = request.FullName;
            profile.About = request.About;
            profile.DateOfBirth = request.DateOfBirth;
            profile.Gender = request.Gender;
            profile.Image = request.Image;

            await _profileRepository.UpdateProfileAsync(profile);

            return new ProfileResponse
            {
                Id = profile.Id,
                Email = profile.Email,
                FullName = profile.FullName,
                RoleId = profile.RoleId,
                RoleName = profile.Role?.Name,
                About = profile.About,
                DateOfBirth = profile.DateOfBirth,
                Gender = profile.Gender,
                Image = profile.Image
            };
        }
    }

    public interface IProfileService
    {
        Task<ProfileResponse> GetProfileAsync(int profileId);
        Task<ProfileResponse> UpdateProfileAsync(int profileId, UpdateProfileRequest request);
    }
}
```

```x-csharp
using System.Collections.Generic;
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using EShoppingZone.Models;
using EShoppingZone.Repositories;

namespace EShoppingZone.Services
{
    public class AddressService : IAddressService
    {
        private readonly IAddressRepository _addressRepository;

        public AddressService(IAddressRepository addressRepository)
        {
            _addressRepository = addressRepository;
        }

        public async Task<List<AddressDto>> GetAddressesAsync(int profileId)
        {
            var addresses = await _addressRepository.GetAddressesByProfileIdAsync(profileId);
            return addresses.Select(a => new AddressDto
            {
                Id = a.Id,
                HouseNumber = a.HouseNumber,
                StreetName = a.StreetName,
                ColonyName = a.ColonyName,
                City = a.City,
                State = a.State,
                Pincode = a.Pincode
            }).ToList();
        }

        public async Task<AddressDto> GetAddressByIdAsync(int id, int profileId)
        {
            var address = await _addressRepository.GetAddressByIdAsync(id, profileId);
            if (address == null)
                throw new Exception("Address not found or unauthorized.");

            return new AddressDto
            {
                Id = address.Id,
                HouseNumber = address.HouseNumber,
                StreetName = address.StreetName,
                ColonyName = address.ColonyName,
                City = address.City,
                State = address.State,
                Pincode = address.Pincode
            };
        }

        public async Task<AddressDto> CreateAddressAsync(CreateAddressRequest request, int profileId)
        {
            var address = new Address
            {
                HouseNumber = request.HouseNumber,
                StreetName = request.StreetName,
                ColonyName = request.ColonyName,
                City = request.City,
                State = request.State,
                Pincode = request.Pincode,
                ProfileId = profileId
            };

            var createdAddress = await _addressRepository.CreateAddressAsync(address);
            return new AddressDto
            {
                Id = createdAddress.Id,
                HouseNumber = createdAddress.HouseNumber,
                StreetName = createdAddress.StreetName,
                ColonyName = createdAddress.ColonyName,
                City = createdAddress.City,
                State = createdAddress.State,
                Pincode = createdAddress.Pincode
            };
        }

        public async Task<AddressDto> UpdateAddressAsync(int id, UpdateAddressRequest request, int profileId)
        {
            var address = await _addressRepository.GetAddressByIdAsync(id, profileId);
            if (address == null)
                throw new Exception("Address not found or unauthorized.");

            address.HouseNumber = request.HouseNumber;
            address.StreetName = request.StreetName;
            address.ColonyName = request.ColonyName;
            address.City = request.City;
            address.State = request.State;
            address.Pincode = request.Pincode;

            var updatedAddress = await _addressRepository.UpdateAddressAsync(address);
            return new AddressDto
            {
                Id = updatedAddress.Id,
                HouseNumber = updatedAddress.HouseNumber,
                StreetName = updatedAddress.StreetName,
                ColonyName = updatedAddress.ColonyName,
                City = updatedAddress.City,
                State = updatedAddress.State,
                Pincode = updatedAddress.Pincode
            };
        }

        public async Task<bool> DeleteAddressAsync(int id, int profileId)
        {
            return await _addressRepository.DeleteAddressAsync(id, profileId);
        }
    }

    public interface IAddressService
    {
        Task<List<AddressDto>> GetAddressesAsync(int profileId);
        Task<AddressDto> GetAddressByIdAsync(int id, int profileId);
        Task<AddressDto> CreateAddressAsync(CreateAddressRequest request, int profileId);
        Task<AddressDto> UpdateAddressAsync(int id, UpdateAddressRequest request, int profileId);
        Task<bool> DeleteAddressAsync(int id, int profileId);
    }
}
```

#### 6. Controllers (Controllers/AuthController.cs, ProfileController.cs, AddressController.cs)
Controllers for authentication, profile, and address endpoints.

```x-csharp
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using EShoppingZone.Services;

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

        [HttpPost("register")]
        public async Task<IActionResult> Register([FromBody] RegisterRequest request)
        {
            try
            {
                var response = await _authService.RegisterAsync(request);
                return Ok(response);
            }
            catch (Exception ex)
            {
                return BadRequest(new { Message = ex.Message });
            }
        }

        [HttpPost("login")]
        public async Task<IActionResult> Login([FromBody] LoginRequest request)
        {
            try
            {
                var response = await _authService.LoginAsync(request);
                return Ok(response);
            }
            catch (Exception ex)
            {
                return BadRequest(new { Message = ex.Message });
            }
        }
    }
}
```

```x-csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using EShoppingZone.Services;

namespace EShoppingZone.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    [Authorize]
    public class ProfileController : ControllerBase
    {
        private readonly IProfileService _profileService;

        public ProfileController(IProfileService profileService)
        {
            _profileService = profileService;
        }

        private int GetUserId()
        {
            return int.Parse(User.FindFirst(ClaimTypes.NameIdentifier)?.Value);
        }

        [HttpGet]
        public async Task<IActionResult> GetProfile()
        {
            try
            {
                var profile = await _profileService.GetProfileAsync(GetUserId());
                return Ok(profile);
            }
            catch (Exception ex)
            {
                return BadRequest(new { Message = ex.Message });
            }
        }

        [HttpPut]
        public async Task<IActionResult> UpdateProfile([FromBody] UpdateProfileRequest request)
        {
            try
            {
                var profile = await _profileService.UpdateProfileAsync(GetUserId(), request);
                return Ok(profile);
            }
            catch (Exception ex)
            {
                return BadRequest(new { Message = ex.Message });
            }
        }
    }
}
```

```x-csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;
using System.Threading.Tasks;
using EShoppingZone.DTOs;
using EShoppingZone.Services;

namespace EShoppingZone.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    [Authorize]
    public class AddressController : ControllerBase
    {
        private readonly IAddressService _addressService;

        public AddressController(IAddressService addressService)
        {
            _addressService = addressService;
        }

        private int GetUserId()
        {
            return int.Parse(User.FindFirst(ClaimTypes.NameIdentifier)?.Value);
        }

        [HttpGet]
        public async Task<IActionResult> GetAddresses()
        {
            try
            {
                var addresses = await _addressService.GetAddressesAsync(GetUserId());
                return Ok(addresses);
            }
            catch (Exception ex)
            {
                return BadRequest(new { Message = ex.Message });
            }
        }

        [HttpGet("{id}")]
        public async Task<IActionResult> GetAddress(int id)
        {
            try
            {
                var address = await _addressService.GetAddressByIdAsync(id, GetUserId());
                return Ok(address);
            }
            catch (Exception ex)
            {
                return BadRequest(new { Message = ex.Message });
            }
        }

        [HttpPost]
        public async Task<IActionResult> CreateAddress([FromBody] CreateAddressRequest request)
        {
            try
            {
                var address = await _addressService.CreateAddressAsync(request, GetUserId());
                return CreatedAtAction(nameof(GetAddress), new { id = address.Id }, address);
            }
            catch (Exception ex)
            {
                return BadRequest(new { Message = ex.Message });
            }
        }

        [HttpPut("{id}")]
        public async Task<IActionResult> UpdateAddress(int id, [FromBody] UpdateAddressRequest request)
        {
            try
            {
                var address = await _addressService.UpdateAddressAsync(id, request, GetUserId());
                return Ok(address);
            }
            catch (Exception ex)
            {
                return BadRequest(new { Message = ex.Message });
            }
        }

        [HttpDelete("{id}")]
        public async Task<IActionResult> DeleteAddress(int id)
        {
            try
            {
                var result = await _addressService.DeleteAddressAsync(id, GetUserId());
                if (!result)
                    return NotFound();
                return NoContent();
            }
            catch (Exception ex)
            {
                return BadRequest(new { Message = ex.Message });
            }
        }
    }
}
```

#### 7. Configuration (appsettings.json)
JWT and database connection settings.

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=EShoppingZoneDb;Trusted_Connection=True;MultipleActiveResultSets=true"
  },
  "Jwt": {
    "Key": "YourSecretKeyHere1234567890",
    "Issuer": "EShoppingZone",
    "Audience": "EShoppingZoneUsers"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  },
  "AllowedHosts": "*"
}
```

#### 8. Program Setup (Program.cs)
Configures services, JWT, and Identity.

```x-csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using System.Text;
using EShoppingZone.Data;
using EShoppingZone.Models;
using EShoppingZone.Repositories;
using EShoppingZone.Services;

var builder = WebApplication.CreateBuilder(args);

// Add services to the container.
builder.Services.AddControllers();

// Configure DbContext
builder.Services.AddDbContext<EShoppingZoneDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Configure Identity
builder.Services.AddIdentity<UserProfile, IdentityRole<int>>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequireLowercase = true;
    options.Password.RequireUppercase = true;
    options.Password.RequireNonAlphanumeric = false;
    options.Password.RequiredLength = 6;
})
.AddEntityFrameworkStores<EShoppingZoneDbContext>()
.AddDefaultTokenProviders();

// Configure JWT Authentication
var jwtSettings = builder.Configuration.GetSection("Jwt");
var key = Encoding.UTF8.GetBytes(jwtSettings["Key"]);

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuer = true,
        ValidateAudience = true,
        ValidateLifetime = true,
        ValidateIssuerSigningKey = true,
        ValidIssuer = jwtSettings["Issuer"],
        ValidAudience = jwtSettings["Audience"],
        IssuerSigningKey = new SymmetricSecurityKey(key)
    };
});

// Register Repositories and Services
builder.Services.AddScoped<IProfileRepository, ProfileRepository>();
builder.Services.AddScoped<IAddressRepository, AddressRepository>();
builder.Services.AddScoped<IAuthService, AuthService>();
builder.Services.AddScoped<IProfileService, ProfileService>();
builder.Services.AddScoped<IAddressService, AddressService>();

// Add Swagger
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Configure the HTTP request pipeline.
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

#### 9. Migration Script (Data/Migrations/InitialCreate.cs)
This is a sample migration to create the database schema. (Note: Actual migration files are generated by EFazionali

```x-csharp
using Microsoft.EntityFrameworkCore.Migrations;

namespace EShoppingZone.Data.Migrations
{
    public partial class InitialCreate : Migration
    {
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.CreateTable(
                name: "Roles",
                columns: table => new
                {
                    RoleId = table.Column<int>(nullable: false)
                        .Annotation("SqlServer:Identity", "1, 1"),
                    Name = table.Column<string>(maxLength: 50, nullable: false)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_Roles", x => x.RoleId);
                });

            migrationBuilder.CreateTable(
                name: "Profiles",
                columns: table => new
                {
                    Id = table.Column<int>(nullable: false)
                        .Annotation("SqlServer:Identity", "1, 1"),
                    UserName = table.Column<string>(maxLength: 256, nullable: true),
                    NormalizedUserName = table.Column<string>(maxLength: 256, nullable: true),
                    Email = table.Column<string>(maxLength: 256, nullable: true),
                    NormalizedEmail = table.Column<string>(maxLength: 256, nullable: true),
                    EmailConfirmed = table.Column<bool>(nullable: false),
                    PasswordHash = table.Column<string>(nullable: true),
                    SecurityStamp = table.Column<string>(nullable: true),
                    ConcurrencyStamp = table.Column<string>(nullable: true),
                    PhoneNumber = table.Column<string>(nullable: true),
                    PhoneNumberConfirmed = table.Column<bool>(nullable: false),
                    TwoFactorEnabled = table.Column<bool>(nullable: false),
                    LockoutEnd = table.Column<DateTimeOffset>(nullable: true),
                    LockoutEnabled = table.Column<bool>(nullable: false),
                    AccessFailedCount = table.Column<int>(nullable: false),
                    FullName = table.Column<string>(maxLength: 100, nullable: false),
                    RoleId = table.Column<int>(nullable: false),
                    About = table.Column<string>(nullable: true),
                    DateOfBirth = table.Column<DateTime>(nullable: true),
                    Gender = table.Column<string>(nullable: true),
                    Image = table.Column<string>(nullable: true)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_Profiles", x => x.Id);
                    table.ForeignKey(
                        name: "FK_Profiles_Roles_RoleId",
                        column: x => x.RoleId,
                        principalTable: "Roles",
                        principalColumn: "RoleId",
                        onDelete: ReferentialAction.Restrict);
                });

            migrationBuilder.CreateTable(
                name: "Addresses",
                columns: table => new
                {
                    Id = table.Column<int>(nullable: false)
                        .Annotation("SqlServer:Identity", "1, 1"),
                    HouseNumber = table.Column<string>(nullable: false),
                    StreetName = table.Column<string>(nullable: false),
                    ColonyName = table.Column<string>(nullable: false),
                    City = table.Column<string>(nullable: false),
                    State = table.Column<string>(nullable: false),
                    Pincode = table.Column<string>(nullable: false),
                    ProfileId = table.Column<int>(nullable: false)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_Addresses", x => x.Id);
                    table.ForeignKey(
                        name: "FK_Addresses_Profiles_ProfileId",
                        column: x => x.ProfileId,
                        principalTable: "Profiles",
                        principalColumn: "Id",
                        onDelete: ReferentialAction.Cascade);
                });

            migrationBuilder.CreateIndex(
                name: "IX_Addresses_ProfileId",
                table: "Addresses",
                column: "ProfileId");

            migrationBuilder.CreateIndex(
                name: "IX_Profiles_RoleId",
                table: "Profiles",
                column: "RoleId");

            migrationBuilder.CreateIndex(
                name: "IX_Roles_Name",
                table: "Roles",
                column: "Name",
                unique: true);
        }

        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropTable(name: "Addresses");
            migrationBuilder.DropTable(name: "Profiles");
            migrationBuilder.DropTable(name: "Roles");
        }
    }
}
```

### Testing Steps
1. **Setup Database**:
   - Update the connection string in `appsettings.json` to point to your SQL Server.
   - Run migrations:
     ```bash
     dotnet ef migrations add InitialCreate
     dotnet ef database update
     ```
   - Seed a role (e.g., `RoleId: 1, Name: "User"`) in the `Roles` table.

2. **Run the API**:
   - Start the application:
     ```bash
     dotnet run
     ```
   - Access Swagger UI at `https://localhost:5001/swagger` (or your configured port).

3. **Test Endpoints**:
   - **Register**:
     ```http
     POST /api/Auth/register
     Content-Type: application/json
     {
       "email": "test@example.com",
       "fullName": "Test User",
       "roleId": 1,
       "password": "Password123",
       "about": "Test user bio",
       "dateOfBirth": "1990-01-01",
       "gender": "Male",
       "image": "profile.jpg"
     }
     ```
     Response: `{ "token": "...", "fullName": "Test User", "id": 1 }`

   - **Login**:
     ```http
     POST /api/Auth/login
     Content-Type: application/json
     {
       "email": "test@example.com",
       "password": "Password123"
     }
     ```
     Response: `{ "token": "...", "fullName": "Test User", "id": 1 }`

   - **Get Profile**:
     ```http
     GET /api/Profile
     Authorization: Bearer <token>
     ```
     Response: Profile details.

   - **Update Profile**:
     ```http
     PUT /api/Profile
     Authorization: Bearer <token>
     Content-Type: application/json
     {
       "fullName": "Updated User",
       "about": "Updated bio",
       "dateOfBirth": "1990-01-01",
       "gender": "Male",
       "image": "updated.jpg"
     }
     ```
     Response: Updated profile details.

   - **Create Address**:
     ```http
     POST /api/Address
     Authorization: Bearer <token>
     Content-Type: application/json
     {
       "houseNumber": "123",
       "streetName": "Main St",
       "colonyName": "Downtown",
       "city": "New York",
       "state": "NY",
       "pincode": "10001"
     }
     ```
     Response: Created address details.

   - **Get Addresses**:
     ```http
     GET /api/Address
     Authorization: Bearer <token>
     ```
     Response: List of user’s addresses.

   - **Update Address**:
     ```http
     PUT /api/Address/1
     Authorization: Bearer <token>
     Content-Type: application/json
     {
       "houseNumber": "456",
       "streetName": "Broadway",
       "colonyName": "Midtown",
       "city": "New York",
       "state": "NY",
       "pincode": "10002"
     }
     ```
     Response: Updated address details.

   - **Delete Address**:
     ```http
     DELETE /api/Address/1
     Authorization: Bearer <token>
     ```
     Response: 204 No Content.

### Suggestions for Improvement
1. **Error Handling**:
   - Implement global exception middleware for consistent error responses.
   - Add validation for `RoleId` existence during registration.
2. **Security**:
   - Store sensitive data (e.g., JWT key) in environment variables or Azure Key Vault.
   - Add rate limiting to prevent brute-force attacks on login.
3. **Performance**:
   - Add pagination for `GET /api/Address` if users have many addresses.
   - Use async/await consistently (already implemented).
4. **Logging**:
   - Integrate a logging framework (e.g., Serilog) for better diagnostics.
5. **Testing**:
   - Write unit tests for services and repositories using xUnit or NUnit.
   - Test JWT token validation and unauthorized access scenarios.
6. **Documentation**:
   - Enhance Swagger with XML comments for better API documentation.
   - Provide Postman collection for easier testing.

This implementation ensures that all CRUD operations for addresses and profile updates are restricted to the logged-in user, with `ProfileId` extracted from the JWT token’s `NameIdentifier` claim. Let me know if you need further clarification or additional features!
