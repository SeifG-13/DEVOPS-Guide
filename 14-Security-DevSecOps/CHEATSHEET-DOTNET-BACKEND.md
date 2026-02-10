# DevSecOps Cheat Sheet for .NET Backend Engineers

## Quick Reference - .NET Security

### OWASP Top 10 for .NET

| Risk | .NET Mitigation |
|------|-----------------|
| Injection | Parameterized queries, EF Core |
| Broken Auth | ASP.NET Core Identity, JWT validation |
| Sensitive Data | Data Protection API, HTTPS |
| XXE | Disable DTD processing |
| Broken Access Control | Authorization policies |
| Security Misconfig | Secure defaults, remove debug |
| XSS | Output encoding, CSP headers |
| Insecure Deserialization | System.Text.Json, type validation |
| Vulnerable Components | NuGet audit, Dependabot |
| Logging Failures | Serilog, structured logging |

---

## Input Validation & Sanitization

### Model Validation
```csharp
public class CreateUserRequest
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    [RegularExpression(@"^[a-zA-Z\s]+$")]
    public string Name { get; set; }

    [Required]
    [EmailAddress]
    public string Email { get; set; }

    [Required]
    [MinLength(8)]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])[A-Za-z\d@$!%*?&]{8,}$",
        ErrorMessage = "Password must contain uppercase, lowercase, number, and special character")]
    public string Password { get; set; }
}
```

### FluentValidation
```csharp
public class CreateUserValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .Must(NotBeDisposableEmail).WithMessage("Disposable emails not allowed");

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(8)
            .Matches("[A-Z]").WithMessage("Must contain uppercase")
            .Matches("[a-z]").WithMessage("Must contain lowercase")
            .Matches("[0-9]").WithMessage("Must contain number")
            .Matches("[^a-zA-Z0-9]").WithMessage("Must contain special character");
    }

    private bool NotBeDisposableEmail(string email)
    {
        var disposableDomains = new[] { "tempmail.com", "throwaway.com" };
        var domain = email.Split('@').LastOrDefault();
        return !disposableDomains.Contains(domain);
    }
}
```

---

## SQL Injection Prevention

### Entity Framework (Safe)
```csharp
// SAFE - Parameterized by default
var user = await _context.Users
    .Where(u => u.Email == email)
    .FirstOrDefaultAsync();

// SAFE - FromSqlInterpolated
var users = await _context.Users
    .FromSqlInterpolated($"SELECT * FROM Users WHERE Email = {email}")
    .ToListAsync();
```

### Dangerous Patterns (AVOID)
```csharp
// DANGEROUS - SQL Injection vulnerable
var query = $"SELECT * FROM Users WHERE Email = '{email}'";
var users = await _context.Users.FromSqlRaw(query).ToListAsync();

// SAFE alternative
var users = await _context.Users
    .FromSqlRaw("SELECT * FROM Users WHERE Email = {0}", email)
    .ToListAsync();
```

---

## Authentication & Authorization

### JWT Configuration
```csharp
// Program.cs
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
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"])),
            ClockSkew = TimeSpan.Zero  // No tolerance for expiry
        };
    });
```

### Authorization Policies
```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("MinimumAge", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));

    options.AddPolicy("OwnerOnly", policy =>
        policy.Requirements.Add(new ResourceOwnerRequirement()));
});

// Controller
[Authorize(Policy = "AdminOnly")]
public IActionResult AdminDashboard() { }
```

---

## Secure Headers

### Security Headers Middleware
```csharp
// Program.cs
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
    context.Response.Headers.Add("Content-Security-Policy",
        "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'");
    context.Response.Headers.Add("Permissions-Policy",
        "accelerometer=(), camera=(), geolocation=(), microphone=()");

    await next();
});

// Or use NWebsec
app.UseXContentTypeOptions();
app.UseXfo(options => options.Deny());
app.UseXXssProtection(options => options.EnabledWithBlockMode());
app.UseCsp(options => options
    .DefaultSources(s => s.Self())
    .ScriptSources(s => s.Self()));
```

### CORS Configuration
```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("Production", policy =>
    {
        policy.WithOrigins("https://myapp.com", "https://api.myapp.com")
              .AllowAnyMethod()
              .AllowAnyHeader()
              .AllowCredentials();
    });
});

// Don't use in production!
// policy.AllowAnyOrigin()  // DANGEROUS with credentials
```

---

## Data Protection

### Encrypt Sensitive Data
```csharp
public class EncryptionService
{
    private readonly IDataProtector _protector;

    public EncryptionService(IDataProtectionProvider provider)
    {
        _protector = provider.CreateProtector("MyApp.SensitiveData");
    }

    public string Encrypt(string plainText)
    {
        return _protector.Protect(plainText);
    }

    public string Decrypt(string encryptedText)
    {
        return _protector.Unprotect(encryptedText);
    }
}
```

### Password Hashing
```csharp
// Using ASP.NET Core Identity (recommended)
var hasher = new PasswordHasher<User>();
var hashedPassword = hasher.HashPassword(user, password);
var result = hasher.VerifyHashedPassword(user, hashedPassword, password);

// Or BCrypt
var hash = BCrypt.Net.BCrypt.HashPassword(password, workFactor: 12);
var isValid = BCrypt.Net.BCrypt.Verify(password, hash);
```

---

## Secrets Management

### User Secrets (Development)
```bash
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:Default" "Server=..."
dotnet user-secrets set "Jwt:Secret" "my-secret-key"
```

### Azure Key Vault (Production)
```csharp
builder.Configuration.AddAzureKeyVault(
    new Uri("https://mykeyvault.vault.azure.net/"),
    new DefaultAzureCredential());

// Access secrets
var connectionString = builder.Configuration["ConnectionStrings--Default"];
```

### Environment Variables
```csharp
// Never hardcode
var secret = Environment.GetEnvironmentVariable("JWT_SECRET");

// Or via configuration
builder.Configuration.AddEnvironmentVariables();
var secret = builder.Configuration["Jwt:Secret"];
```

---

## Dependency Scanning

### NuGet Audit
```bash
# Check for vulnerabilities
dotnet list package --vulnerable

# Check for outdated packages
dotnet list package --outdated

# In CI/CD
dotnet restore
dotnet list package --vulnerable --include-transitive
```

### GitHub Dependabot
```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
```

---

## Interview Q&A

### Q1: How do you prevent SQL injection in .NET?
**A:**
- Use Entity Framework (parameterized by default)
- Use `FromSqlInterpolated` not `FromSqlRaw` with concatenation
- Never concatenate user input into SQL strings
- Validate and sanitize input

### Q2: How do you store passwords securely?
**A:**
- Use `PasswordHasher<T>` from ASP.NET Core Identity
- Or BCrypt/Argon2
- Never store plain text
- Never use MD5/SHA1 for passwords

### Q3: How do you secure API endpoints?
**A:**
```csharp
// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)...

// Authorization
[Authorize(Policy = "AdminOnly")]
public class AdminController : ControllerBase { }

// Rate limiting
builder.Services.AddRateLimiter(options => ...);

// Input validation
[Required, StringLength(100)]
public string Name { get; set; }
```

### Q4: How do you handle sensitive data in logs?
**A:**
```csharp
// Never log sensitive data
_logger.LogInformation("User {UserId} logged in", userId);  // Good
_logger.LogInformation("Password: {Password}", password);   // BAD!

// Use destructuring with exclusions
public record User([property: NotLogged] string Password);
```

### Q5: How do you implement HTTPS in .NET?
**A:**
```csharp
// Force HTTPS
app.UseHttpsRedirection();
app.UseHsts();

// In production
builder.WebHost.UseKestrel(options =>
{
    options.ListenAnyIP(443, listenOptions =>
    {
        listenOptions.UseHttps("certificate.pfx", "password");
    });
});
```

### Q6: What is the Data Protection API?
**A:** .NET's built-in encryption for:
- Session cookies
- Anti-forgery tokens
- Custom encryption needs

```csharp
var protector = _provider.CreateProtector("purpose");
var encrypted = protector.Protect(data);
var decrypted = protector.Unprotect(encrypted);
```

---

## Security Checklist

### Code
- [ ] Input validation on all endpoints
- [ ] Parameterized queries only
- [ ] Password hashing (not encryption)
- [ ] Secrets in Key Vault/User Secrets
- [ ] No sensitive data in logs

### Configuration
- [ ] HTTPS enforced
- [ ] Security headers set
- [ ] CORS properly configured
- [ ] Debug disabled in production
- [ ] Error details hidden

### Dependencies
- [ ] Regular `dotnet list package --vulnerable`
- [ ] Dependabot/Snyk enabled
- [ ] Minimal packages installed
- [ ] Transitive dependencies checked

### Authentication/Authorization
- [ ] Strong JWT validation
- [ ] Token expiration enforced
- [ ] Authorization policies defined
- [ ] Rate limiting enabled

---

## GitHub Actions Security Scan

```yaml
name: Security Scan

on: [push, pull_request]

jobs:
  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '8.0.x'

      - name: Restore
        run: dotnet restore

      - name: Check vulnerable packages
        run: |
          dotnet list package --vulnerable --include-transitive 2>&1 | tee vulnerabilities.txt
          if grep -q "has the following vulnerable packages" vulnerabilities.txt; then
            echo "Vulnerable packages found!"
            exit 1
          fi

      - name: Security scan with Snyk
        uses: snyk/actions/dotnet@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
```

---

## Best Practices

1. **Validate all input** - Never trust user data
2. **Use parameterized queries** - EF Core by default
3. **Hash passwords properly** - BCrypt or Identity
4. **Manage secrets securely** - Key Vault, never in code
5. **Enable HTTPS** - UseHttpsRedirection, HSTS
6. **Set security headers** - CSP, X-Frame-Options
7. **Scan dependencies** - Regular vulnerability checks
8. **Log securely** - No sensitive data in logs
