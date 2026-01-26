---
name: jwt-authentication
description: "Configures JWT Bearer authentication for .NET APIs. Includes token generation, validation, refresh tokens, and user context extraction from claims."
version: 1.0.0
language: C#
framework: .NET 8+
dependencies: Microsoft.AspNetCore.Authentication.JwtBearer, System.IdentityModel.Tokens.Jwt
---

# JWT Authentication Setup

## Overview

This skill implements JWT (JSON Web Token) authentication for .NET APIs:

- **Token generation** - Create access and refresh tokens
- **Token validation** - Validate incoming tokens
- **User context** - Extract user info from claims
- **Refresh flow** - Handle token refresh

## Quick Reference

| Component | Purpose | Location |
|-----------|---------|----------|
| `IJwtService` | Token generation interface | Application/Abstractions |
| `JwtService` | Token generation implementation | Infrastructure/Authentication |
| `JwtBearerOptionsSetup` | Configure JWT validation | Infrastructure/Authentication |
| `IUserContext` | Current user info | Application/Abstractions |
| `UserContext` | Extract from HttpContext | Infrastructure/Authentication |

---

## Authentication Structure

```
/Application/Abstractions/
├── Authentication/
│   ├── IJwtService.cs
│   ├── IUserContext.cs
│   └── TokenResponse.cs

/Infrastructure/
├── Authentication/
│   ├── JwtService.cs
│   ├── JwtBearerOptionsSetup.cs
│   ├── UserContext.cs
│   └── AuthenticationExtensions.cs
```

---

## Template: JWT Configuration Options

```csharp
// src/{name}.infrastructure/Authentication/JwtOptions.cs
namespace {name}.infrastructure.authentication;

public sealed class JwtOptions
{
    public const string SectionName = "Jwt";

    public string Issuer { get; init; } = string.Empty;
    public string Audience { get; init; } = string.Empty;
    public string SecretKey { get; init; } = string.Empty;
    public int AccessTokenExpirationMinutes { get; init; } = 60;
    public int RefreshTokenExpirationDays { get; init; } = 7;
}
```

### appsettings.json

```json
{
  "Jwt": {
    "Issuer": "your-app-name",
    "Audience": "your-app-name",
    "SecretKey": "your-secret-key-at-least-32-characters-long-for-security",
    "AccessTokenExpirationMinutes": 60,
    "RefreshTokenExpirationDays": 7
  }
}
```

---

## Template: JWT Service Interface

```csharp
// src/{name}.application/Abstractions/Authentication/IJwtService.cs
using {name}.domain.users;

namespace {name}.application.abstractions.authentication;

public interface IJwtService
{
    /// <summary>
    /// Generates access and refresh tokens for a user
    /// </summary>
    TokenResponse GenerateTokens(User user, IEnumerable<string> roles);

    /// <summary>
    /// Generates access and refresh tokens with custom claims
    /// </summary>
    TokenResponse GenerateTokens(
        Guid userId,
        string email,
        IEnumerable<string> roles,
        IDictionary<string, string>? additionalClaims = null);

    /// <summary>
    /// Validates a refresh token and returns user ID if valid
    /// </summary>
    Guid? ValidateRefreshToken(string refreshToken);

    /// <summary>
    /// Generates a new access token from a valid refresh token
    /// </summary>
    TokenResponse? RefreshAccessToken(string refreshToken);
}
```

---

## Template: Token Response

```csharp
// src/{name}.application/Abstractions/Authentication/TokenResponse.cs
namespace {name}.application.abstractions.authentication;

public sealed record TokenResponse(
    string AccessToken,
    string RefreshToken,
    DateTime AccessTokenExpiration,
    DateTime RefreshTokenExpiration);
```

---

## Template: JWT Service Implementation

```csharp
// src/{name}.infrastructure/Authentication/JwtService.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;
using {name}.application.abstractions.authentication;
using {name}.domain.users;

namespace {name}.infrastructure.authentication;

internal sealed class JwtService : IJwtService
{
    private readonly JwtOptions _options;
    private readonly SigningCredentials _signingCredentials;
    private readonly JwtSecurityTokenHandler _tokenHandler;

    public JwtService(IOptions<JwtOptions> options)
    {
        _options = options.Value;
        
        var securityKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(_options.SecretKey));
        
        _signingCredentials = new SigningCredentials(
            securityKey,
            SecurityAlgorithms.HmacSha256);
        
        _tokenHandler = new JwtSecurityTokenHandler();
    }

    public TokenResponse GenerateTokens(User user, IEnumerable<string> roles)
    {
        return GenerateTokens(
            user.Id,
            user.Email.Value,
            roles,
            new Dictionary<string, string>
            {
                { "name", user.Name },
                { "organization_id", user.OrganizationId.ToString() }
            });
    }

    public TokenResponse GenerateTokens(
        Guid userId,
        string email,
        IEnumerable<string> roles,
        IDictionary<string, string>? additionalClaims = null)
    {
        var accessTokenExpiration = DateTime.UtcNow
            .AddMinutes(_options.AccessTokenExpirationMinutes);
        
        var refreshTokenExpiration = DateTime.UtcNow
            .AddDays(_options.RefreshTokenExpirationDays);

        var accessToken = GenerateAccessToken(
            userId,
            email,
            roles,
            additionalClaims,
            accessTokenExpiration);

        var refreshToken = GenerateRefreshToken(userId, refreshTokenExpiration);

        return new TokenResponse(
            accessToken,
            refreshToken,
            accessTokenExpiration,
            refreshTokenExpiration);
    }

    private string GenerateAccessToken(
        Guid userId,
        string email,
        IEnumerable<string> roles,
        IDictionary<string, string>? additionalClaims,
        DateTime expiration)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, userId.ToString()),
            new(JwtRegisteredClaimNames.Email, email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat, 
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64)
        };

        // Add roles
        foreach (var role in roles)
        {
            claims.Add(new Claim(ClaimTypes.Role, role));
        }

        // Add additional claims
        if (additionalClaims is not null)
        {
            foreach (var (key, value) in additionalClaims)
            {
                claims.Add(new Claim(key, value));
            }
        }

        var token = new JwtSecurityToken(
            issuer: _options.Issuer,
            audience: _options.Audience,
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: expiration,
            signingCredentials: _signingCredentials);

        return _tokenHandler.WriteToken(token);
    }

    private string GenerateRefreshToken(Guid userId, DateTime expiration)
    {
        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, userId.ToString()),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new("token_type", "refresh")
        };

        var token = new JwtSecurityToken(
            issuer: _options.Issuer,
            audience: _options.Audience,
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: expiration,
            signingCredentials: _signingCredentials);

        return _tokenHandler.WriteToken(token);
    }

    public Guid? ValidateRefreshToken(string refreshToken)
    {
        try
        {
            var principal = _tokenHandler.ValidateToken(
                refreshToken,
                GetValidationParameters(),
                out var validatedToken);

            if (validatedToken is not JwtSecurityToken jwtToken)
            {
                return null;
            }

            // Verify it's a refresh token
            var tokenType = principal.FindFirst("token_type")?.Value;
            if (tokenType != "refresh")
            {
                return null;
            }

            var userIdClaim = principal.FindFirst(JwtRegisteredClaimNames.Sub)?.Value;
            
            return Guid.TryParse(userIdClaim, out var userId) ? userId : null;
        }
        catch
        {
            return null;
        }
    }

    public TokenResponse? RefreshAccessToken(string refreshToken)
    {
        var userId = ValidateRefreshToken(refreshToken);
        
        if (userId is null)
        {
            return null;
        }

        // In a real application, you would:
        // 1. Look up the user from the database
        // 2. Check if the refresh token is still valid (not revoked)
        // 3. Get the user's current roles and claims
        // For now, we generate new tokens with minimal claims
        
        return GenerateTokens(
            userId.Value,
            string.Empty,  // Would get from database
            Array.Empty<string>());  // Would get from database
    }

    private TokenValidationParameters GetValidationParameters()
    {
        return new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = _options.Issuer,
            ValidateAudience = true,
            ValidAudience = _options.Audience,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(_options.SecretKey)),
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero
        };
    }
}
```

---

## Template: JWT Bearer Options Setup

```csharp
// src/{name}.infrastructure/Authentication/JwtBearerOptionsSetup.cs
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;

namespace {name}.infrastructure.authentication;

internal sealed class JwtBearerOptionsSetup 
    : IConfigureNamedOptions<JwtBearerOptions>
{
    private readonly JwtOptions _jwtOptions;

    public JwtBearerOptionsSetup(IOptions<JwtOptions> jwtOptions)
    {
        _jwtOptions = jwtOptions.Value;
    }

    public void Configure(JwtBearerOptions options)
    {
        Configure(JwtBearerDefaults.AuthenticationScheme, options);
    }

    public void Configure(string? name, JwtBearerOptions options)
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = _jwtOptions.Issuer,
            
            ValidateAudience = true,
            ValidAudience = _jwtOptions.Audience,
            
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(_jwtOptions.SecretKey)),
            
            ValidateLifetime = true,
            ClockSkew = TimeSpan.Zero,  // No tolerance for expiration
            
            // Ensure we get the user ID from the token
            NameClaimType = "sub",
            RoleClaimType = "role"
        };

        options.Events = new JwtBearerEvents
        {
            OnAuthenticationFailed = context =>
            {
                if (context.Exception is SecurityTokenExpiredException)
                {
                    context.Response.Headers.Append(
                        "Token-Expired",
                        "true");
                }
                return Task.CompletedTask;
            },
            OnChallenge = context =>
            {
                // Custom response for 401
                return Task.CompletedTask;
            },
            OnForbidden = context =>
            {
                // Custom response for 403
                return Task.CompletedTask;
            }
        };
    }
}
```

---

## Template: User Context Interface

```csharp
// src/{name}.application/Abstractions/Authentication/IUserContext.cs
namespace {name}.application.abstractions.authentication;

public interface IUserContext
{
    /// <summary>
    /// Current authenticated user's ID
    /// </summary>
    Guid UserId { get; }

    /// <summary>
    /// Current user's email
    /// </summary>
    string Email { get; }

    /// <summary>
    /// Current user's organization ID
    /// </summary>
    Guid? OrganizationId { get; }

    /// <summary>
    /// Current user's roles
    /// </summary>
    IReadOnlyList<string> Roles { get; }

    /// <summary>
    /// Check if user is authenticated
    /// </summary>
    bool IsAuthenticated { get; }

    /// <summary>
    /// Check if user has a specific role
    /// </summary>
    bool IsInRole(string role);

    /// <summary>
    /// Get a custom claim value
    /// </summary>
    string? GetClaimValue(string claimType);
}
```

---

## Template: User Context Implementation

```csharp
// src/{name}.infrastructure/Authentication/UserContext.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using Microsoft.AspNetCore.Http;
using {name}.application.abstractions.authentication;

namespace {name}.infrastructure.authentication;

internal sealed class UserContext : IUserContext
{
    private readonly IHttpContextAccessor _httpContextAccessor;

    public UserContext(IHttpContextAccessor httpContextAccessor)
    {
        _httpContextAccessor = httpContextAccessor;
    }

    private ClaimsPrincipal? User => _httpContextAccessor.HttpContext?.User;

    public bool IsAuthenticated => User?.Identity?.IsAuthenticated ?? false;

    public Guid UserId
    {
        get
        {
            var userIdClaim = User?.FindFirst(JwtRegisteredClaimNames.Sub)?.Value
                ?? User?.FindFirst(ClaimTypes.NameIdentifier)?.Value;

            return Guid.TryParse(userIdClaim, out var userId)
                ? userId
                : throw new InvalidOperationException("User ID not found in claims");
        }
    }

    public string Email
    {
        get
        {
            return User?.FindFirst(JwtRegisteredClaimNames.Email)?.Value
                ?? User?.FindFirst(ClaimTypes.Email)?.Value
                ?? string.Empty;
        }
    }

    public Guid? OrganizationId
    {
        get
        {
            var orgIdClaim = User?.FindFirst("organization_id")?.Value;
            return Guid.TryParse(orgIdClaim, out var orgId) ? orgId : null;
        }
    }

    public IReadOnlyList<string> Roles
    {
        get
        {
            return User?.FindAll(ClaimTypes.Role)
                .Select(c => c.Value)
                .ToList()
                ?? new List<string>();
        }
    }

    public bool IsInRole(string role)
    {
        return User?.IsInRole(role) ?? false;
    }

    public string? GetClaimValue(string claimType)
    {
        return User?.FindFirst(claimType)?.Value;
    }
}
```

---

## Template: Authentication Registration

```csharp
// src/{name}.infrastructure/Authentication/AuthenticationExtensions.cs
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;
using {name}.application.abstractions.authentication;

namespace {name}.infrastructure.authentication;

public static class AuthenticationExtensions
{
    public static IServiceCollection AddJwtAuthentication(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        // Bind JWT options
        services.Configure<JwtOptions>(
            configuration.GetSection(JwtOptions.SectionName));

        // Register JWT Bearer authentication
        services
            .AddAuthentication(options =>
            {
                options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
                options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
                options.DefaultScheme = JwtBearerDefaults.AuthenticationScheme;
            })
            .AddJwtBearer();

        // Configure JWT Bearer options
        services.ConfigureOptions<JwtBearerOptionsSetup>();

        // Register services
        services.AddHttpContextAccessor();
        services.AddScoped<IJwtService, JwtService>();
        services.AddScoped<IUserContext, UserContext>();

        return services;
    }
}
```

---

## Template: Login Command

```csharp
// src/{name}.application/Users/Login/LoginUserCommand.cs
using {name}.application.abstractions.authentication;
using {name}.application.abstractions.messaging;
using {name}.domain.abstractions;
using {name}.domain.users;

namespace {name}.application.users.login;

public sealed record LoginUserCommand(
    string Email,
    string Password) : ICommand<TokenResponse>;

internal sealed class LoginUserCommandHandler
    : ICommandHandler<LoginUserCommand, TokenResponse>
{
    private readonly IUserRepository _userRepository;
    private readonly IPasswordHasher _passwordHasher;
    private readonly IJwtService _jwtService;
    private readonly IRoleRepository _roleRepository;

    public LoginUserCommandHandler(
        IUserRepository userRepository,
        IPasswordHasher passwordHasher,
        IJwtService jwtService,
        IRoleRepository roleRepository)
    {
        _userRepository = userRepository;
        _passwordHasher = passwordHasher;
        _jwtService = jwtService;
        _roleRepository = roleRepository;
    }

    public async Task<Result<TokenResponse>> Handle(
        LoginUserCommand request,
        CancellationToken cancellationToken)
    {
        // Find user by email
        var user = await _userRepository.GetByEmailAsync(
            request.Email,
            cancellationToken);

        if (user is null)
        {
            return Result.Failure<TokenResponse>(UserErrors.InvalidCredentials);
        }

        // Verify password
        if (!_passwordHasher.Verify(request.Password, user.PasswordHash))
        {
            return Result.Failure<TokenResponse>(UserErrors.InvalidCredentials);
        }

        // Check if user is active
        if (!user.IsActive)
        {
            return Result.Failure<TokenResponse>(UserErrors.AccountDeactivated);
        }

        // Get user roles
        var roles = await _roleRepository.GetRolesByUserIdAsync(
            user.Id,
            cancellationToken);

        // Generate tokens
        var tokenResponse = _jwtService.GenerateTokens(
            user,
            roles.Select(r => r.Name));

        return tokenResponse;
    }
}
```

---

## Template: Refresh Token Command

```csharp
// src/{name}.application/Users/RefreshToken/RefreshTokenCommand.cs
using {name}.application.abstractions.authentication;
using {name}.application.abstractions.messaging;
using {name}.domain.abstractions;
using {name}.domain.users;

namespace {name}.application.users.refreshtoken;

public sealed record RefreshTokenCommand(
    string RefreshToken) : ICommand<TokenResponse>;

internal sealed class RefreshTokenCommandHandler
    : ICommandHandler<RefreshTokenCommand, TokenResponse>
{
    private readonly IJwtService _jwtService;
    private readonly IUserRepository _userRepository;
    private readonly IRoleRepository _roleRepository;

    public RefreshTokenCommandHandler(
        IJwtService jwtService,
        IUserRepository userRepository,
        IRoleRepository roleRepository)
    {
        _jwtService = jwtService;
        _userRepository = userRepository;
        _roleRepository = roleRepository;
    }

    public async Task<Result<TokenResponse>> Handle(
        RefreshTokenCommand request,
        CancellationToken cancellationToken)
    {
        // Validate refresh token
        var userId = _jwtService.ValidateRefreshToken(request.RefreshToken);

        if (userId is null)
        {
            return Result.Failure<TokenResponse>(UserErrors.InvalidRefreshToken);
        }

        // Get user
        var user = await _userRepository.GetByIdAsync(
            userId.Value,
            cancellationToken);

        if (user is null || !user.IsActive)
        {
            return Result.Failure<TokenResponse>(UserErrors.InvalidRefreshToken);
        }

        // Get current roles
        var roles = await _roleRepository.GetRolesByUserIdAsync(
            user.Id,
            cancellationToken);

        // Generate new tokens
        var tokenResponse = _jwtService.GenerateTokens(
            user,
            roles.Select(r => r.Name));

        return tokenResponse;
    }
}
```

---

## Template: Auth Controller

```csharp
// src/{name}.api/Controllers/Auth/AuthController.cs
using MediatR;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using {name}.application.users.login;
using {name}.application.users.refreshtoken;

namespace {name}.api.Controllers.Auth;

[ApiController]
[Route("api/v1/auth")]
public class AuthController : ControllerBase
{
    private readonly ISender _sender;

    public AuthController(ISender sender)
    {
        _sender = sender;
    }

    [HttpPost("login")]
    [AllowAnonymous]
    public async Task<IActionResult> Login(
        [FromBody] LoginRequest request,
        CancellationToken cancellationToken)
    {
        var command = new LoginUserCommand(request.Email, request.Password);

        var result = await _sender.Send(command, cancellationToken);

        if (result.IsFailure)
        {
            return Unauthorized(result.Error);
        }

        return Ok(result.Value);
    }

    [HttpPost("refresh")]
    [AllowAnonymous]
    public async Task<IActionResult> RefreshToken(
        [FromBody] RefreshTokenRequest request,
        CancellationToken cancellationToken)
    {
        var command = new RefreshTokenCommand(request.RefreshToken);

        var result = await _sender.Send(command, cancellationToken);

        if (result.IsFailure)
        {
            return Unauthorized(result.Error);
        }

        return Ok(result.Value);
    }

    [HttpGet("me")]
    [Authorize]
    public IActionResult GetCurrentUser()
    {
        // User info is available from HttpContext.User
        return Ok(new
        {
            UserId = User.FindFirst("sub")?.Value,
            Email = User.FindFirst("email")?.Value,
            Roles = User.FindAll("role").Select(c => c.Value)
        });
    }
}

public sealed record LoginRequest(string Email, string Password);
public sealed record RefreshTokenRequest(string RefreshToken);
```

---

## Critical Rules

1. **Secret key length** - At least 32 characters for HMAC-SHA256
2. **Store secrets securely** - Use Azure Key Vault, AWS Secrets, etc.
3. **Short access tokens** - 15-60 minutes typical
4. **Longer refresh tokens** - 7-30 days typical
5. **Validate all claims** - Issuer, audience, signature, expiration
6. **No clock skew** - Set `ClockSkew = TimeSpan.Zero`
7. **HTTPS only** - Never transmit tokens over HTTP
8. **Don't store in localStorage** - Use httpOnly cookies for web apps
9. **Revoke refresh tokens** - On logout, password change
10. **Use IUserContext** - Don't access HttpContext directly in handlers

---

## Anti-Patterns to Avoid

```csharp
// ❌ WRONG: Short secret key
"SecretKey": "abc123"  // Too short, insecure!

// ✅ CORRECT: Strong secret key
"SecretKey": "your-secret-key-at-least-32-characters-long-for-security"

// ❌ WRONG: Accessing HttpContext in handler
public class Handler
{
    private readonly IHttpContextAccessor _accessor;
    var userId = _accessor.HttpContext.User.FindFirst("sub");  // Don't!
}

// ✅ CORRECT: Use IUserContext abstraction
public class Handler
{
    private readonly IUserContext _userContext;
    var userId = _userContext.UserId;
}

// ❌ WRONG: Never expiring tokens
expires: DateTime.MaxValue  // Security risk!

// ✅ CORRECT: Short-lived tokens with refresh
expires: DateTime.UtcNow.AddMinutes(60)
```

---

## Related Skills

- `permission-authorization` - Permission-based access control
- `api-controller-generator` - Protected API endpoints
- `dotnet-clean-architecture` - Infrastructure layer setup
