1. ModelsModels/UserProfile.csusing Microsoft.AspNetCore.Identity;
using System.ComponentModel.DataAnnotations;

namespace EShoppingZone.Models
{
    public class UserProfile : IdentityUser<int>
    {
        [Required]
        [StringLength(100)]
        public string FullName { get; set; }

        public string About { get; set; }
        public DateTime? DateOfBirth { get; set; }
        public string Gender { get; set; }
        public string Image { get; set; }

        public List<Address> Addresses { get; set; } = new List<Address>();
    }
}Models/Address.csusing System.ComponentModel.DataAnnotations;
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
}2. DTOsDTOs/AuthDtos.csusing System.ComponentModel.DataAnnotations;

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
}DTOs/ProfileDtos.csusing System.ComponentModel.DataAnnotations;

namespace EShoppingZone.DTOs
{
    public class ProfileResponse
    {
        public int Id { get; set; }
        public string Email { get; set; }
        public string FullName { get; set; }
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
}DTOs/AddressDtos.csusing System.ComponentModel.DataAnnotations;

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
}3. DbContext

3. DbContextData/EShoppingZoneDbContext.csusing Microsoft.AspNetCore.Identity.EntityFrameworkCore;
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
                .HasMany(up => up.Addresses)
                .WithOne(a => a.Profile)
                .HasForeignKey(a => a.ProfileId)
                .OnDelete(DeleteBehavior.Cascade);

            // Address configuration
            modelBuilder.Entity<Address>()
                .HasKey(a => a.Id);
        }
    }
}4. RepositoriesRepositories/ProfileRepository.csusing Microsoft.EntityFrameworkCore;
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
}Repositories/AddressRepository.csusing Microsoft.EntityFrameworkCore;
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
}5. Services

5. ServicesServices/AuthService.csusing Microsoft.AspNetCore.Identity;
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
                new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
                new Claim("FullName", user.FullName)
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
}Services/ProfileService.csusing System.Threading.Tasks;
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
}Services/AddressService.csusing System.Collections.Generic;
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
}6. Controllers

6. ControllersControllers/AuthController.csusing Microsoft.AspNetCore.Mvc;
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
}Controllers/ProfileController.csusing Microsoft.AspNetCore.Authorization;
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
}Controllers/AddressController.csusing Microsoft.AspNetCore.Authorization;
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
}7. Configurationappsettings.json{
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
}8. Program SetupProgram.csusing Microsoft.AspNetCore.Authentication.JwtBearer;
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

app.Run();9. Migration ScriptData/Migrations/InitialCreate.csusing Microsoft.EntityFrameworkCore.Migrations;

namespace EShoppingZone.Data.Migrations
{
    public partial class InitialCreate : Migration
    {
        protected override void Up(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.CreateTable(
                name: "Profiles",
                columns: table => new
                {
                    Id = table.Column<int>(type: "int", nullable: false)
                        .Annotation("SqlServer:Identity", "1, 1"),
                    UserName = table.Column<string>(type: "nvarchar(256)", maxLength: 256, nullable: true),
                    NormalizedUserName = table.Column<string>(type: "nvarchar(256)", maxLength: 256, nullable: true),
                    Email = table.Column<string>(type: "nvarchar(256)", maxLength: 256, nullable: true),
                    NormalizedEmail = table.Column<string>(type: "nvarchar(256)", maxLength: 256, nullable: true),
                    EmailConfirmed = table.Column<bool>(type: "bit", nullable: false),
                    PasswordHash = table.Column<string>(type: "nvarchar(max)", nullable: true),
                    SecurityStamp = table.Column<string>(type: "nvarchar(max)", nullable: true),
                    ConcurrencyStamp = table.Column<string>(type: "nvarchar(max)", nullable: true),
                    PhoneNumber = table.Column<string>(type: "nvarchar(max)", nullable: true),
                    PhoneNumberConfirmed = table.Column<bool>(type: "bit", nullable: false),
                    TwoFactorEnabled = table.Column<bool>(type: "bit", nullable: false),
                    LockoutEnd = table.Column<DateTimeOffset>(type: "datetimeoffset", nullable: true),
                    LockoutEnabled = table.Column<bool>(type: "bit", nullable: false),
                    AccessFailedCount = table.Column<int>(type: "int", nullable: false),
                    FullName = table.Column<string>(type: "nvarchar(100)", maxLength: 100, nullable: false),
                    About = table.Column<string>(type: "nvarchar(max)", nullable: true),
                    DateOfBirth = table.Column<DateTime>(type: "datetime2", nullable: true),
                    Gender = table.Column<string>(type: "nvarchar(max)", nullable: true),
                    Image = table.Column<string>(type: "nvarchar(max)", nullable: true)
                },
                constraints: table =>
                {
                    table.PrimaryKey("PK_Profiles", x => x.Id);
                });

            migrationBuilder.CreateTable(
                name: "Addresses",
                columns: table => new
                {
                    Id = table.Column<int>(type: "int", nullable: false)
                        .Annotation("SqlServer:Identity", "1, 1"),
                    HouseNumber = table.Column<string>(type: "nvarchar(max)", nullable: false),
                    StreetName = table.Column<string>(type: "nvarchar(max)", nullable: false),
                    ColonyName = table.Column<string>(type: "nvarchar(max)", nullable: false),
                    City = table.Column<string>(type: "nvarchar(max)", nullable: false),
                    State = table.Column<string>(type: "nvarchar(max)", nullable: false),
                    Pincode = table.Column<string>(type: "nvarchar(max)", nullable: false),
                    ProfileId = table.Column<int>(type: "int", nullable: false)
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
        }

        protected override void Down(MigrationBuilder migrationBuilder)
        {
            migrationBuilder.DropTable(name: "Addresses");
            migrationBuilder.DropTable(name: "Profiles");
        }
    }
}