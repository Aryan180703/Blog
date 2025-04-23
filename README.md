# Blog


using EShoppingZone.Interface;
using EShoppingZone.DTOs;
using EShoppingZone.Profile;
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

        public async Task<ResponseDTO<ProfileResponse>> CreateProfileAsync(CreateProfileRequest request)
        {
            // Validate required fields and RoleId
            if (string.IsNullOrEmpty(request.FullName) || string.IsNullOrEmpty(request.EmailId) ||
                string.IsNullOrEmpty(request.Password) || request.RoleId <= 0)
                return new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = "FullName, EmailId, Password, and a valid RoleId are required.",
                    Data = null
                };

            // Check if RoleId exists
            var role = await _repository.GetRoleAsync(request.RoleId);
            if (role == null)
                return new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = $"Role with ID {request.RoleId} does not exist.",
                    Data = null
                };

            var profile = new UserProfile
            {
                FullName = request.FullName,
                EmailId = request.EmailId,
                MobileNumber = request.MobileNumber,
                Password = request.Password, // Hash in production
                RoleId = request.RoleId,
                About = request.About,
                DateOfBirth = request.DateOfBirth,
                Gender = request.Gender,
                Image = request.Image
            };

            await _repository.AddProfileAsync(profile);

            var response = new ProfileResponse
            {
                ProfileId = profile.ProfileId,
                FullName = profile.FullName,
                EmailId = profile.EmailId,
                MobileNumber = profile.MobileNumber,
                About = profile.About,
                DateOfBirth = profile.DateOfBirth,
                Gender = profile.Gender,
                Image = profile.Image,
                RoleId = profile.RoleId,
                RoleName = role.Name,
                Addresses = new List<AddressResponse>()
            };

            return new ResponseDTO<ProfileResponse>
            {
                Success = true,
                Message = "Profile created successfully.",
                Data = response
            };
        }

        public async Task<ResponseDTO<ProfileResponse>> GetProfileAsync(int profileId)
        {
            var profile = await _repository.GetProfileAsync(profileId);
            if (profile == null)
                return new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = "Profile not found.",
                    Data = null
                };

            var response = new ProfileResponse
            {
                ProfileId = profile.ProfileId,
                FullName = profile.FullName,
                EmailId = profile.EmailId,
                MobileNumber = profile.MobileNumber,
                About = profile.About,
                DateOfBirth = profile.DateOfBirth,
                Gender = profile.Gender,
                Image = profile.Image,
                RoleId = profile.RoleId,
                RoleName = profile.Role?.Name,
                Addresses = profile.Addresses?.Select(a => new AddressResponse
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

        public async Task<ResponseDTO<ProfileResponse>> UpdateProfileAsync(int profileId, UpdateProfileRequest request)
        {
            var profile = await _repository.GetProfileAsync(profileId);
            if (profile == null)
                return new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = "Profile not found.",
                    Data = null
                };

            profile.FullName = request.FullName ?? profile.FullName;
            profile.EmailId = request.EmailId ?? profile.EmailId;
            profile.MobileNumber = request.MobileNumber != 0 ? request.MobileNumber : profile.MobileNumber;
            profile.About = request.About ?? profile.About;
            profile.DateOfBirth = request.DateOfBirth ?? profile.DateOfBirth;
            profile.Gender = request.Gender ?? profile.Gender;
            profile.Image = request.Image ?? profile.Image;

            await _repository.UpdateProfileAsync(profile);

            var response = new ProfileResponse
            {
                ProfileId = profile.ProfileId,
                FullName = profile.FullName,
                EmailId = profile.EmailId,
                MobileNumber = profile.MobileNumber,
                About = profile.About,
                DateOfBirth = profile.DateOfBirth,
                Gender = profile.Gender,
                Image = profile.Image,
                RoleId = profile.RoleId,
                RoleName = profile.Role?.Name,
                Addresses = profile.Addresses?.Select(a => new AddressResponse
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

        public async Task<ResponseDTO<AddressResponse>> CreateAddressAsync(int profileId, CreateAddressRequest request)
        {
            var profile = await _repository.GetProfileAsync(profileId);
            if (profile == null)
                return new ResponseDTO<AddressResponse>
                {
                    Success = false,
                    Message = "Profile not found.",
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
                ProfileId = profileId
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

        public async Task<ResponseDTO<AddressResponse>> UpdateAddressAsync(int profileId, int addressId, UpdateAddressRequest request)
        {
            var address = await _repository.GetAddressAsync(addressId);
            if (address == null || address.ProfileId != profileId)
                return new ResponseDTO<AddressResponse>
                {
                    Success = false,
                    Message = "Address not found or does not belong to profile.",
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






using EShoppingZone.Context;
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

        public async Task AddProfileAsync(UserProfile profile)
        {
            await _context.Profiles.AddAsync(profile);
            await _context.SaveChangesAsync();
        }

        public async Task<UserProfile> GetProfileAsync(int profileId)
        {
            return await _context.Profiles
                .Include(p => p.Role)
                .Include(p => p.Addresses)
                .FirstOrDefaultAsync(p => p.ProfileId == profileId);
        }

        public async Task UpdateProfileAsync(UserProfile profile)
        {
            _context.Profiles.Update(profile);
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







using EShoppingZone.Interface;
using EShoppingZone.DTOs;
using Microsoft.AspNetCore.Mvc;
using System.Threading.Tasks;
using System.Collections.Generic;
using Microsoft.EntityFrameworkCore;
using System.Linq;

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

        [HttpPost("customer")]
        public async Task<IActionResult> CreateProfile([FromBody] CreateProfileRequest request)
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

            try
            {
                var result = await _service.CreateProfileAsync(request);
                if (!result.Success)
                    return BadRequest(result);

                return CreatedAtAction(nameof(GetProfile), new { profileId = result.Data.ProfileId }, result);
            }
            catch (DbUpdateException)
            {
                return BadRequest(new ResponseDTO<ProfileResponse>
                {
                    Success = false,
                    Message = "Failed to create profile due to database error (e.g., invalid RoleId).",
                    Data = null
                });
            }
        }

        [HttpGet("{profileId}")]
        public async Task<IActionResult> GetProfile(int profileId)
        {
            var result = await _service.GetProfileAsync(profileId);
            if (!result.Success)
                return NotFound(result);

            return Ok(result);
        }

        [HttpPut("{profileId}")]
        public async Task<IActionResult> UpdateProfile(int profileId, [FromBody] UpdateProfileRequest request)
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

            var result = await _service.UpdateProfileAsync(profileId, request);
            if (!result.Success)
                return NotFound(result);

            return Ok(result);
        }

        [HttpPost("{profileId}/addresses")]
        public async Task<IActionResult> CreateAddress(int profileId, [FromBody] CreateAddressRequest request)
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

            var result = await _service.CreateAddressAsync(profileId, request);
            if (!result.Success)
                return NotFound(result);

            return CreatedAtAction(nameof(GetProfile), new { profileId }, result);
        }

        [HttpPut("{profileId}/addresses/{addressId}")]
        public async Task<IActionResult> UpdateAddress(int profileId, int addressId, [FromBody] UpdateAddressRequest request)
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

            var result = await _service.UpdateAddressAsync(profileId, addressId, request);
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
