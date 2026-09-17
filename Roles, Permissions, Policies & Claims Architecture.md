# Roles, Permissions, Policies & Claims Implementation Guide for HrmsSys
## Clean Monolith (.NET 8/9/10 + Next.js App Router)

> **Architecture Style**: Clean / Onion Architecture Monolith (No Microservices)  
> **Solution Structure**:  
> • **Backend**: `HrmsSys/backend/` (`Domain`, `Application`, `Infrastructure`, `Api`)  
> • **Frontend**: `HrmsSys/frontend/` (Next.js App Router with `src/modules/`)  
> **Primary Callback / Public Form Strategy**: **Option B — Magic Links & Temporary Signed Tokens** (Deep Dive included)

---

## 1. Architectural Blueprint for `HrmsSys`

Unlike a distributed microservices setup, your project is a **Modular Clean Architecture Monolith**. All modules (`attendance`, `leave`, `payroll`, `recruitment`, `loans`, `letters`, `documents`, `approvals`, `companies`, etc.) live within a single solution, sharing a unified database context and memory space.

```
HrmsSys/
├── backend/
│   ├── Api/                     --> Hosts Controllers, Filters, Middleware, Program.cs
│   │   ├── Controllers/         --> [Authorize(Policy = "...")] gated endpoints
│   │   ├── Filters/             --> [RequireMagicLink], [ApiKeyAuthorize] filters
│   │   ├── Middleware/          --> Global exception & JWT claim mapping
│   │   └── Program.cs           --> Authentication, Authorization & DI Wiring
│   ├── Application/             --> Core Business Logic, Interfaces, DTOs
│   │   ├── Common/              --> Dynamic Policy Provider, Requirement, Handler
│   │   │   └── Authorization/   --> PermissionRequirement, PolicyProvider, Handler
│   │   ├── DTOs/                --> Requests & Responses (Roles, Perms, Auth, Forms)
│   │   └── Services/            --> Application Services & Interfaces
│   ├── Domain/                  --> Enterprise Entities & Business Rules
│   │   ├── Entities/            --> ApplicationUser, ApplicationRole, Permission, RolePermission, TemporaryAccessLink
│   │   └── Enums/               --> System Roles, Link Types, Statuses
│   └── Infrastructure/          --> EF Core, Database Persistence, JWT Token Generation
│       ├── Persistence/         --> AppDbContext, SeedData, EF Configurations
│       └── Services/            --> TokenService, MagicLinkService
└── frontend/
    └── src/
        ├── app/                 --> (auth), (dashboard) Next.js App Router
        ├── components/          --> HasPermission guards, UI controls
        ├── lib/                 --> Token decoders & API client
        └── modules/             --> Feature modules (leave, payroll, recruitment, etc.)
```

---

## 2. Domain Entities (`backend/Domain/Entities/`)

Create your identity and authorization models in `Domain/Entities/`.

### 2.1. Core Identity & Permission Entities (`Domain/Entities/IdentityEntities.cs`)

```csharp
using Microsoft.AspNetCore.Identity;

namespace Domain.Entities;

// 1. Extended Identity User
public class ApplicationUser : IdentityUser
{
    public string FullName { get; set; } = string.Empty;
    public string? EmployeeCode { get; set; }
    public Guid? CompanyId { get; set; }          // Multi-company / group tenant
    public Guid? DepartmentId { get; set; }
    public Guid? DesignationId { get; set; }
    public bool IsActive { get; set; } = true;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public DateTime? LastLoginAt { get; set; }
}

// 2. Extended Identity Role
public class ApplicationRole : IdentityRole
{
    public string? Description { get; set; }
    public bool IsSystemRole { get; set; } = false;
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    public ApplicationRole() : base() { }
    public ApplicationRole(string roleName) : base(roleName) { }
}

// 3. Permission Catalog Entity (Fine-grained capability)
public class Permission
{
    public int Id { get; set; }
    public string Code { get; set; } = string.Empty;       // e.g. "leave.approve", "payroll.process"
    public string Name { get; set; } = string.Empty;       // e.g. "Approve Leave Requests"
    public string Module { get; set; } = string.Empty;     // e.g. "Leave", "Payroll", "Recruitment"
    public string? Description { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    public ICollection<RolePermission> RolePermissions { get; set; } = [];
}

// 4. Join Entity (Which role possesses which permissions)
public class RolePermission
{
    public string RoleId { get; set; } = string.Empty;
    public ApplicationRole Role { get; set; } = null!;

    public int PermissionId { get; set; }
    public Permission Permission { get; set; } = null!;

    public DateTime AssignedAt { get; set; } = DateTime.UtcNow;
}
```

### 2.2. Entity for Magic Links & Temporary Tokens (`Domain/Entities/TemporaryAccessLink.cs`)

This entity powers **Option B** (candidate pre-onboarding forms, reference checks, external letter signatures, and feedback).

```csharp
namespace Domain.Entities;

public class TemporaryAccessLink
{
    public Guid Id { get; set; } = Guid.NewGuid();
    
    // Cryptographically secure token string or hash
    public string Token { get; set; } = string.Empty;
    
    // Purpose: "CandidatePreOnboarding", "JobApplicationDocument", "ExitClearance", "DocumentSign"
    public string Purpose { get; set; } = string.Empty;
    
    // Target entity reference (e.g. CandidateId, EmployeeId, DocumentId)
    public string ReferenceId { get; set; } = string.Empty;
    
    // Recipient email
    public string RecipientEmail { get; set; } = string.Empty;
    
    public DateTime ExpiresAt { get; set; }
    public bool IsUsed { get; set; } = false;
    public DateTime? UsedAt { get; set; }
    public bool IsRevoked { get; set; } = false;
    
    // Optional JSON payload for form pre-fill or meta configuration
    public string? MetadataJson { get; set; }
    
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

---

## 3. Infrastructure & Persistence (`backend/Infrastructure/Persistence/`)

### 3.1. Database Context (`Infrastructure/Persistence/AppDbContext.cs`)

Configure ASP.NET Core Identity tables alongside your HRMS business tables in `AppDbContext`:

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using Domain.Entities;

namespace Infrastructure.Persistence;

public class AppDbContext(DbContextOptions<AppDbContext> options)
    : IdentityDbContext<ApplicationUser, ApplicationRole, string>(options)
{
    public DbSet<Permission> Permissions => Set<Permission>();
    public DbSet<RolePermission> RolePermissions => Set<RolePermission>();
    public DbSet<TemporaryAccessLink> TemporaryAccessLinks => Set<TemporaryAccessLink>();

    // Add other HRMS DbSets here:
    // public DbSet<Employee> Employees => Set<Employee>();
    // public DbSet<LeaveRequest> LeaveRequests => Set<LeaveRequest>();
    // public DbSet<PayrollRun> PayrollRuns => Set<PayrollRun>();
    // public DbSet<LoanApplication> Loans => Set<LoanApplication>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // Rename standard Identity tables to clean names
        builder.Entity<ApplicationUser>(e => e.ToTable("sys_users"));
        builder.Entity<ApplicationRole>(e => e.ToTable("sys_roles"));
        builder.Entity<IdentityUserRole<string>>(e => e.ToTable("sys_user_roles"));
        builder.Entity<IdentityUserClaim<string>>(e => e.ToTable("sys_user_claims"));
        builder.Entity<IdentityUserLogin<string>>(e => e.ToTable("sys_user_logins"));
        builder.Entity<IdentityUserToken<string>>(e => e.ToTable("sys_user_tokens"));
        builder.Entity<IdentityRoleClaim<string>>(e => e.ToTable("sys_role_claims"));

        // Permissions catalog
        builder.Entity<Permission>(e =>
        {
            e.ToTable("sys_permissions");
            e.HasKey(p => p.Id);
            e.HasIndex(p => p.Code).IsUnique();
            e.Property(p => p.Code).HasMaxLength(100).IsRequired();
            e.Property(p => p.Name).HasMaxLength(150).IsRequired();
            e.Property(p => p.Module).HasMaxLength(100).IsRequired();
        });

        // RolePermission composite join table
        builder.Entity<RolePermission>(e =>
        {
            e.ToTable("sys_role_permissions");
            e.HasKey(rp => new { rp.RoleId, rp.PermissionId });
            
            e.HasOne(rp => rp.Role)
                .WithMany()
                .HasForeignKey(rp => rp.RoleId)
                .OnDelete(DeleteBehavior.Cascade);

            e.HasOne(rp => rp.Permission)
                .WithMany(p => p.RolePermissions)
                .HasForeignKey(rp => rp.PermissionId)
                .OnDelete(DeleteBehavior.Cascade);
        });

        // Temporary Access Links (Option B)
        builder.Entity<TemporaryAccessLink>(e =>
        {
            e.ToTable("sys_temporary_links");
            e.HasKey(l => l.Id);
            e.HasIndex(l => l.Token).IsUnique();
            e.Property(l => l.Purpose).HasMaxLength(80).IsRequired();
            e.Property(l => l.RecipientEmail).HasMaxLength(150).IsRequired();
        });
    }
}
```

---

## 4. The Dynamic Policy Provider Engine (`backend/Application/Common/Authorization/`)

This engine is what gives you zero-boilerplate authorization: **any string in `[Authorize(Policy = "leave.approve")]` works automatically without pre-registering it in `Program.cs`**.

### 4.1. `PermissionRequirement.cs`
Path: `backend/Application/Common/Authorization/PermissionRequirement.cs`

```csharp
using Microsoft.AspNetCore.Authorization;

namespace Application.Common.Authorization;

public class PermissionRequirement(string permission) : IAuthorizationRequirement
{
    public string Permission { get; } = permission;
}
```

### 4.2. `PermissionPolicyProvider.cs`
Path: `backend/Application/Common/Authorization/PermissionPolicyProvider.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.Extensions.Options;

namespace Application.Common.Authorization;

public class PermissionPolicyProvider(IOptions<AuthorizationOptions> options) : IAuthorizationPolicyProvider
{
    private readonly DefaultAuthorizationPolicyProvider _fallback = new(options);

    public bool AllowsCachingPolicies => ((IAuthorizationPolicyProvider)_fallback).AllowsCachingPolicies;

    public Task<AuthorizationPolicy> GetDefaultPolicyAsync() => _fallback.GetDefaultPolicyAsync();
    public Task<AuthorizationPolicy?> GetFallbackPolicyAsync() => _fallback.GetFallbackPolicyAsync();

    public Task<AuthorizationPolicy?> GetPolicyAsync(string policyName)
    {
        // Treat any policyName as an arbitrary permission code requirement
        var policy = new AuthorizationPolicyBuilder()
            .RequireAuthenticatedUser()
            .AddRequirements(new PermissionRequirement(policyName))
            .Build();

        return Task.FromResult<AuthorizationPolicy?>(policy);
    }
}
```

### 4.3. `PermissionAuthorizationHandler.cs`
Path: `backend/Application/Common/Authorization/PermissionAuthorizationHandler.cs`

```csharp
using Microsoft.AspNetCore.Authorization;

namespace Application.Common.Authorization;

public class PermissionAuthorizationHandler : AuthorizationHandler<PermissionRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, 
        PermissionRequirement requirement)
    {
        // 1. Superuser Bypass: The "Admin" role automatically passes every permission check
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // 2. Fine-grained claim check: Does token contain claim ("permission", requirement.Permission)?
        if (context.User.HasClaim(c => c.Type == "permission" && c.Value == requirement.Permission))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

### 4.4. Dependency Injection Extension Method
Path: `backend/Application/Common/Authorization/AuthorizationExtensions.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.Extensions.DependencyInjection;

namespace Application.Common.Authorization;

public static class AuthorizationExtensions
{
    public static IServiceCollection AddDynamicPermissionAuthorization(this IServiceCollection services)
    {
        services.AddSingleton<IAuthorizationPolicyProvider, PermissionPolicyProvider>();
        services.AddScoped<IAuthorizationHandler, PermissionAuthorizationHandler>();
        return services;
    }
}
```

---

## 5. HRMS Permission Catalog & Seeding Strategy

Based on your frontend modules (`admin`, `approvals`, `attendance`, `companies`, `documents`, `employees`, `leave`, `letters`, `loans`, `org-masters`, `payroll`, `recruitment`, `reports`), here is the complete seed catalog.

### 5.1. `SeedData.cs`
Path: `backend/Infrastructure/Persistence/SeedData.cs`

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Domain.Entities;

namespace Infrastructure.Persistence;

public static class SeedData
{
    private static readonly (string Module, string Code, string Name)[] Catalog =
    [
        // Company & Org Masters
        ("Company Setup", "companies.view", "View Company Profiles"),
        ("Company Setup", "companies.manage", "Create/Edit Companies and Branches"),
        ("Org Masters", "orgmasters.view", "View Departments, Designations & Grades"),
        ("Org Masters", "orgmasters.manage", "Manage Departments, Designations & Grades"),
        ("Compliance", "compliance.view", "View Statutory Rules & Compliance"),
        ("Compliance", "compliance.manage", "Manage Statutory Rules & Compliance"),

        // Employees & Directory
        ("Employees", "employees.view", "View Employee Directory & Profiles"),
        ("Employees", "employees.create", "Create New Employee"),
        ("Employees", "employees.edit", "Edit Employee Details"),
        ("Employees", "employees.delete", "Deactivate/Terminate Employee"),

        // Attendance
        ("Attendance", "attendance.view", "View Attendance Logs & Summaries"),
        ("Attendance", "attendance.record", "Punch/Record Attendance"),
        ("Attendance", "attendance.manage", "Shift Scheduling & Punch Overrides"),

        // Leave & Approvals
        ("Leave", "leave.view", "View Leave Applications"),
        ("Leave", "leave.apply", "Apply For Leave (Self & On Behalf)"),
        ("Leave", "leave.approve", "Approve/Reject Leave Requests"),
        ("Leave", "leave.manage", "Configure Leave Quotas and Policies"),
        ("Approvals", "approvals.manage", "Universal Approval Workflow Access"),

        // Payroll
        ("Payroll", "payroll.view", "View Payroll Register"),
        ("Payroll", "payroll.process", "Run Monthly Payroll Computations"),
        ("Payroll", "payroll.pay", "Authorize Salary Payouts"),
        ("Payroll", "payroll.reports", "Generate Salary & Statutory Reports"),

        // Loans & Advances
        ("Loans", "loans.view", "View Employee Loans"),
        ("Loans", "loans.apply", "Apply For Salary Advance/Loan"),
        ("Loans", "loans.approve", "Approve/Reject Loans"),

        // Recruitment & Onboarding
        ("Recruitment", "recruitment.view", "View Job Openings & Candidates"),
        ("Recruitment", "recruitment.manage", "Create Postings & Move Pipeline"),
        ("Recruitment", "recruitment.interview", "Schedule & Evaluate Interviews"),
        ("Recruitment", "recruitment.onboard", "Generate Candidate Pre-Onboarding Links"),

        // Letters & Documents
        ("Letters", "letters.view", "View Letter Templates & Generated Letters"),
        ("Letters", "letters.generate", "Generate Experience/Offer/Promotion Letters"),
        ("Documents", "documents.view", "View Employee Documents"),
        ("Documents", "documents.manage", "Upload/Verify Sensitive Documents"),

        // Reports & System Admin
        ("Reports", "reports.view", "Generate HR, Attendance & Payroll Reports"),
        ("Admin", "users.manage", "Manage User Logins & Status"),
        ("Admin", "roles.manage", "Manage Roles & Permission Assignment")
    ];

    private static readonly Dictionary<string, (string Description, string[] Perms)> RoleMatrix = new()
    {
        ["Admin"] = ("Super Administrator", ["*"]),

        ["HR_Director"] = ("Head of Human Resources", [
            "companies.view", "orgmasters.view", "orgmasters.manage", "compliance.view", "compliance.manage",
            "employees.view", "employees.create", "employees.edit", "employees.delete",
            "attendance.view", "attendance.manage",
            "leave.view", "leave.approve", "leave.manage", "approvals.manage",
            "payroll.view", "payroll.process", "payroll.pay", "payroll.reports",
            "loans.view", "loans.approve",
            "recruitment.view", "recruitment.manage", "recruitment.interview", "recruitment.onboard",
            "letters.view", "letters.generate", "documents.view", "documents.manage",
            "reports.view", "users.manage", "roles.manage"
        ]),

        ["HR_Executive"] = ("HR Operations Specialist", [
            "orgmasters.view", "compliance.view",
            "employees.view", "employees.create", "employees.edit",
            "attendance.view", "attendance.manage",
            "leave.view", "leave.approve",
            "recruitment.view", "recruitment.manage", "recruitment.interview", "recruitment.onboard",
            "letters.view", "letters.generate", "documents.view", "documents.manage",
            "reports.view"
        ]),

        ["Payroll_Specialist"] = ("Payroll & Accounts Officer", [
            "employees.view", "attendance.view", "leave.view",
            "payroll.view", "payroll.process", "payroll.pay", "payroll.reports",
            "loans.view", "loans.approve",
            "reports.view"
        ]),

        ["Department_Manager"] = ("Department Head / Reporting Manager", [
            "employees.view", "attendance.view",
            "leave.view", "leave.approve", "approvals.manage",
            "loans.view", "recruitment.interview", "reports.view"
        ]),

        ["Employee"] = ("Standard Staff / Self-Service", [
            "attendance.record", "leave.apply", "loans.apply"
        ])
    };

    public static async Task SeedAsync(
        AppDbContext db,
        UserManager<ApplicationUser> userManager,
        RoleManager<ApplicationRole> roleManager)
    {
        await db.Database.EnsureCreatedAsync();

        // 1. Seed Permissions Catalog
        foreach (var (module, code, name) in Catalog)
        {
            if (!await db.Permissions.AnyAsync(p => p.Code == code))
            {
                db.Permissions.Add(new Permission { Module = module, Code = code, Name = name });
            }
        }
        await db.SaveChangesAsync();

        // 2. Seed Roles and their Permissions
        var allPerms = await db.Permissions.ToListAsync();
        foreach (var (roleName, (desc, perms)) in RoleMatrix)
        {
            var role = await roleManager.FindByNameAsync(roleName);
            if (role is null)
            {
                role = new ApplicationRole(roleName)
                {
                    Description = desc,
                    IsSystemRole = roleName == "Admin"
                };
                await roleManager.CreateAsync(role);
            }

            var granted = perms.Contains("*")
                ? allPerms
                : allPerms.Where(p => perms.Contains(p.Code)).ToList();

            foreach (var perm in granted)
            {
                if (!await db.RolePermissions.AnyAsync(rp => rp.RoleId == role.Id && rp.PermissionId == perm.Id))
                {
                    db.RolePermissions.Add(new RolePermission { RoleId = role.Id, PermissionId = perm.Id });
                }
            }
        }
        await db.SaveChangesAsync();

        // 3. Seed Default Admin User
        const string adminEmail = "admin@hrms.local";
        if (await userManager.FindByEmailAsync(adminEmail) is null)
        {
            var admin = new ApplicationUser
            {
                UserName = adminEmail,
                Email = adminEmail,
                FullName = "System SuperAdmin",
                EmailConfirmed = true,
                IsActive = true
            };
            await userManager.CreateAsync(admin, "Admin@Hrms2026!");
            await userManager.AddToRoleAsync(admin, "Admin");
        }
    }
}
```

---

## 6. Token Generation & Claims Packaging

### 6.1. `TokenService.cs`
Path: `backend/Infrastructure/Services/TokenService.cs`

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.Configuration;
using Microsoft.IdentityModel.Tokens;
using Domain.Entities;
using Infrastructure.Persistence;

namespace Infrastructure.Services;

public class TokenService(IConfiguration config, AppDbContext db)
{
    public async Task<(string token, DateTime expires)> CreateTokenAsync(
        ApplicationUser user, 
        IList<string> roles)
    {
        var jwt = config.GetSection("Jwt");
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwt["Key"]!));
        var creds = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);
        var expires = DateTime.UtcNow.AddHours(int.Parse(jwt["ExpiryHours"] ?? "8"));

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, user.Id),
            new(JwtRegisteredClaimNames.Email, user.Email!),
            new("fullName", user.FullName),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        };

        // Standard role claims
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        // Multi-tenant & Organization context
        if (user.CompanyId.HasValue)
            claims.Add(new Claim("company_id", user.CompanyId.Value.ToString()));

        if (user.DepartmentId.HasValue)
            claims.Add(new Claim("department_id", user.DepartmentId.Value.ToString()));

        // Resolve all distinct permission codes across all assigned roles
        var roleIds = await db.Roles
            .Where(r => roles.Contains(r.Name!))
            .Select(r => r.Id)
            .ToListAsync();

        var permissions = await db.RolePermissions
            .Where(rp => roleIds.Contains(rp.RoleId))
            .Select(rp => rp.Permission.Code)
            .Distinct()
            .ToListAsync();

        // Pack permission codes as custom claims
        claims.AddRange(permissions.Select(p => new Claim("permission", p)));

        var token = new JwtSecurityToken(
            issuer: jwt["Issuer"],
            audience: jwt["Audience"],
            claims: claims,
            expires: expires,
            signingCredentials: creds);

        return (new JwtSecurityTokenHandler().WriteToken(token), expires);
    }
}
```

---

## 7. `Program.cs` Configuration (`backend/Api/Program.cs`)

Here is how everything wires together in your API layer. Notice the registration order:

```csharp
using System.Text;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;
using Application.Common.Authorization;
using Domain.Entities;
using Infrastructure.Persistence;
using Infrastructure.Services;

var builder = WebApplication.CreateBuilder(args);

// 1. Database Context
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection"))); 
    // Or UseNpgsql for PostgreSQL

// 2. Identity Configuration
builder.Services.AddIdentity<ApplicationUser, ApplicationRole>(options =>
{
    options.Password.RequireDigit = true;
    options.Password.RequiredLength = 8;
    options.Password.RequireNonAlphanumeric = false;
})
.AddEntityFrameworkStores<AppDbContext>()
.AddDefaultTokenProviders();

// 3. JWT Authentication
var jwt = builder.Configuration.GetSection("Jwt");
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.RequireHttpsMetadata = false;
        options.SaveToken = true;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = jwt["Issuer"],
            ValidAudience = jwt["Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwt["Key"]!))
        };
    });

// 4. Default Authorization (Default-Deny Fallback Policy)
builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});

// 5. Dynamic Permission Policy Provider (Zero-boilerplate)
// CRITICAL: Must be registered AFTER AddAuthorization()
builder.Services.AddDynamicPermissionAuthorization();

// 6. Application & Infrastructure Services
builder.Services.AddScoped<TokenService>();
builder.Services.AddScoped<MagicLinkService>();

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// Seed database on startup
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    var userMgr = scope.ServiceProvider.GetRequiredService<UserManager<ApplicationUser>>();
    var roleMgr = scope.ServiceProvider.GetRequiredService<RoleManager<ApplicationRole>>();
    await SeedData.SeedAsync(db, userMgr, roleMgr);
}

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

---

## 8. Controller & Action Level Enforcement

### 8.1. Administrative Actions vs. Self-Service
Path: `backend/Api/Controllers/LeaveController.cs`

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace Api.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize] // All actions require authentication by default
public class LeaveController : ControllerBase
{
    private string CurrentUserId => User.FindFirstValue(ClaimTypes.NameIdentifier)!;

    // ── 1. Admin/Manager Gated Action ──
    [Authorize(Policy = "leave.approve")]
    [HttpPut("{id:guid}/approve")]
    public async Task<IActionResult> ApproveLeave(Guid id)
    {
        // Only callers with "leave.approve" claim OR "Admin" role can reach this line
        return Ok(new { message = $"Leave application {id} approved." });
    }

    [Authorize(Policy = "leave.view")]
    [HttpGet]
    public async Task<IActionResult> GetAllLeaves()
    {
        return Ok(new { message = "Listing all company leave applications." });
    }

    // ── 2. Employee Self-Service Action ──
    // Any authenticated staff member can apply for their OWN leave
    [HttpPost("my-leaves")]
    public async Task<IActionResult> ApplyMyLeave([FromBody] ApplyLeaveDto dto)
    {
        // Enforce ownership: caller can only apply for their own user id
        dto.ApplicantUserId = CurrentUserId;
        return Ok(new { message = "Leave submitted successfully." });
    }
}
```

---

## 9. Public Form Filing & Callbacks: Detailed Implementation

You specifically requested options for **callbacks pushed and public form filling**, highlighting **Option B (Magic Links / Temporary Signed Tokens)** as your preferred approach.

Here are the 3 complete options:

---

### Option B (Preferred): Magic Links & Temporary Signed Tokens

#### When to Use:
* **Candidate Pre-Onboarding Form**: Selected candidate receives a link via email: `https://hrms.yourcompany.com/onboard?token=...` to submit personal details, bank accounts, and photo ID before having an employee account.
* **External Reference Check Questionnaire**: Recommender fills in a rating form without needing an HRMS login.
* **Exit Interview / Clearance Form**: Resigned employee completes exit forms after their active account has been disabled.
* **Document Signing**: An applicant or contractor reviews and signs an offer letter.

#### Why it is Superior:
1. **No Account Needed**: The recipient does not need an `ApplicationUser` account or password.
2. **Cryptographically Secure**: Cannot be guessed, brute-forced, or reused if marked single-use.
3. **Time-Bound**: Automatically expires after $N$ hours or days.
4. **Scoped**: The token only grants access to one specific form/purpose (e.g. `CandidatePreOnboarding`), not any other part of the system.

---

#### 1. Implementation of `MagicLinkService`
Path: `backend/Infrastructure/Services/MagicLinkService.cs`

```csharp
using System.Security.Cryptography;
using Microsoft.EntityFrameworkCore;
using Domain.Entities;
using Infrastructure.Persistence;

namespace Infrastructure.Services;

public class MagicLinkService(AppDbContext db)
{
    /// <summary>
    /// Generates a tamper-proof, time-bound temporary access link.
    /// </summary>
    public async Task<(string Token, string FullUrl)> GenerateMagicLinkAsync(
        string purpose, 
        string referenceId, 
        string recipientEmail, 
        int validDays = 7, 
        string? metadataJson = null)
    {
        // Generate high-entropy 256-bit cryptographically secure token
        var randomBytes = new byte[32];
        RandomNumberGenerator.Fill(randomBytes);
        var token = Convert.ToHexString(randomBytes).ToLowerInvariant();

        var link = new TemporaryAccessLink
        {
            Token = token,
            Purpose = purpose,
            ReferenceId = referenceId,
            RecipientEmail = recipientEmail,
            ExpiresAt = DateTime.UtcNow.AddDays(validDays),
            MetadataJson = metadataJson
        };

        db.TemporaryAccessLinks.Add(link);
        await db.SaveChangesAsync();

        // Target URL matching your Next.js frontend route
        var fullUrl = $"https://hrms.company.com/public/forms/{purpose.ToLowerInvariant()}?token={token}";
        return (token, fullUrl);
    }

    /// <summary>
    /// Validates and marks the magic link as used.
    /// </summary>
    public async Task<TemporaryAccessLink?> ValidateAndConsumeTokenAsync(string token, string expectedPurpose)
    {
        var link = await db.TemporaryAccessLinks
            .FirstOrDefaultAsync(l => l.Token == token && l.Purpose == expectedPurpose);

        if (link is null) return null; // Token does not exist or purpose mismatch
        if (link.IsRevoked) return null; // Link was manually cancelled by HR
        if (link.IsUsed) return null; // Single-use token already consumed
        if (DateTime.UtcNow > link.ExpiresAt) return null; // Token expired

        // Mark as consumed
        link.IsUsed = true;
        link.UsedAt = DateTime.UtcNow;
        await db.SaveChangesAsync();

        return link;
    }

    /// <summary>
    /// Validates token without consuming it (useful for initial form render/hydration).
    /// </summary>
    public async Task<TemporaryAccessLink?> InspectTokenAsync(string token, string expectedPurpose)
    {
        var link = await db.TemporaryAccessLinks
            .AsNoTracking()
            .FirstOrDefaultAsync(l => l.Token == token && l.Purpose == expectedPurpose);

        if (link is null || link.IsRevoked || link.IsUsed || DateTime.UtcNow > link.ExpiresAt)
            return null;

        return link;
    }
}
```

---

#### 2. Reusable Action Filter for Controller Endpoints
Path: `backend/Api/Filters/RequireMagicLinkAttribute.cs`

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Infrastructure.Services;

namespace Api.Filters;

[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class RequireMagicLinkAttribute(string purpose, bool consumeOnSuccess = true) : Attribute, IAsyncActionFilter
{
    private const string TokenHeaderName = "X-Magic-Token";
    private const string TokenQueryParam = "token";

    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var httpContext = context.HttpContext;
        var magicLinkService = httpContext.RequestServices.GetRequiredService<MagicLinkService>();

        // Extract token from Header or Query String
        string? token = null;
        if (httpContext.Request.Headers.TryGetValue(TokenHeaderName, out var headerVal))
        {
            token = headerVal.ToString();
        }
        else if (httpContext.Request.Query.TryGetValue(TokenQueryParam, out var queryVal))
        {
            token = queryVal.ToString();
        }

        if (string.IsNullOrWhiteSpace(token))
        {
            context.Result = new UnauthorizedObjectResult(new { message = "Magic link token is missing." });
            return;
        }

        // Validate token
        var link = consumeOnSuccess 
            ? await magicLinkService.ValidateAndConsumeTokenAsync(token, purpose)
            : await magicLinkService.InspectTokenAsync(token, purpose);

        if (link is null)
        {
            context.Result = new UnauthorizedObjectResult(new { 
                message = "The magic link is invalid, expired, or has already been submitted." 
            });
            return;
        }

        // Store validated link entity in HttpContext items for the action to read
        httpContext.Items["ValidatedMagicLink"] = link;

        await next();
    }
}
```

---

#### 3. Controller Endpoints for Candidate Form Filing
Path: `backend/Api/Controllers/CandidateOnboardingController.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Api.Filters;
using Domain.Entities;
using Infrastructure.Services;

namespace Api.Controllers;

[ApiController]
[Route("api/public/onboarding")]
public class CandidateOnboardingController(MagicLinkService magicService) : ControllerBase
{
    // ── STEP 1: HR Generates the Link (Authenticated Action) ──
    [Authorize(Policy = "recruitment.onboard")]
    [HttpPost("generate-link")]
    public async Task<IActionResult> GenerateCandidateLink([FromBody] GenerateOnboardingLinkDto req)
    {
        var (token, url) = await magicService.GenerateMagicLinkAsync(
            purpose: "CandidatePreOnboarding",
            referenceId: req.CandidateId.ToString(),
            recipientEmail: req.CandidateEmail,
            validDays: 7
        );

        // You can send 'url' via email here using your email sender service
        return Ok(new { token, url, expiresAt = DateTime.UtcNow.AddDays(7) });
    }

    // ── STEP 2: Candidate's Browser Fetches Pre-fill Data (Read-only inspection) ──
    [AllowAnonymous]
    [RequireMagicLink("CandidatePreOnboarding", consumeOnSuccess: false)]
    [HttpGet("form-info")]
    public IActionResult GetFormContext()
    {
        var link = (TemporaryAccessLink)HttpContext.Items["ValidatedMagicLink"]!;
        return Ok(new
        {
            candidateId = link.ReferenceId,
            email = link.RecipientEmail,
            expiresAt = link.ExpiresAt
        });
    }

    // ── STEP 3: Candidate Submits Form (Atomic consumption) ──
    [AllowAnonymous]
    [RequireMagicLink("CandidatePreOnboarding", consumeOnSuccess: true)]
    [HttpPost("submit")]
    public async Task<IActionResult> SubmitPreOnboardingForm([FromBody] CandidatePreOnboardingFormDto dto)
    {
        var link = (TemporaryAccessLink)HttpContext.Items["ValidatedMagicLink"]!;
        var candidateId = Guid.Parse(link.ReferenceId);

        // Process data (save personal details, bank info, emergency contact, upload documents)
        // await onboardingService.SavePreOnboardingDataAsync(candidateId, dto);

        return Ok(new { message = "Onboarding details saved successfully. Welcome aboard!" });
    }
}
```

---

### Option A: Fully Public Open Form (`[AllowAnonymous]`)

Use this for your **Public Careers Job Application Form** (where anyone can apply from your careers webpage).

Path: `backend/Api/Controllers/PublicCareersController.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace Api.Controllers;

[ApiController]
[Route("api/public/careers")]
public class PublicCareersController : ControllerBase
{
    /// <summary>
    /// Public job application endpoint.
    /// Uses [AllowAnonymous] to bypass the default fallback authorization.
    /// </summary>
    [AllowAnonymous]
    [HttpPost("apply")]
    public async Task<IActionResult> ApplyForJob([FromBody] PublicJobApplicationDto req)
    {
        // 1. In production, verify Google reCAPTCHA or Cloudflare Turnstile token
        // if (!await captchaValidator.VerifyAsync(req.CaptchaToken))
        //     return BadRequest("Bot verification failed.");

        // 2. Persist application record in Recruitment module
        return Ok(new { message = "Application received successfully." });
    }
}
```

---

### Option C: Inbound Hardware Webhook & Biometric Punch Push

Use this when physical **biometric punch machines** (e.g. ZKTeco, Hikvision, Essl) or external payment gateways push real-time callbacks to your backend.

#### 1. Custom Action Filter for Machine Authentication
Path: `backend/Api/Filters/ApiKeyAuthorizeAttribute.cs`

```csharp
using System.Security.Cryptography;
using System.Text;
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.Configuration;
using Microsoft.Extensions.DependencyInjection;

namespace Api.Filters;

public class ApiKeyAuthorizeAttribute(string configKey = "Integrations:BiometricApiKey") 
    : Attribute, IAsyncActionFilter
{
    private const string HeaderKey = "X-API-KEY";

    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var config = context.HttpContext.RequestServices.GetRequiredService<IConfiguration>();
        var expectedKey = config[configKey];

        if (!context.HttpContext.Request.Headers.TryGetValue(HeaderKey, out var providedKey) ||
            string.IsNullOrWhiteSpace(providedKey) ||
            !FixedTimeEquals(expectedKey!, providedKey!))
        {
            context.Result = new UnauthorizedObjectResult(new { message = "Invalid Machine API Key." });
            return;
        }

        await next();
    }

    private static bool FixedTimeEquals(string strA, string strB)
    {
        var bytesA = Encoding.UTF8.GetBytes(strA);
        var bytesB = Encoding.UTF8.GetBytes(strB);
        return CryptographicOperations.FixedTimeEquals(bytesA, bytesB);
    }
}
```

#### 2. Punch Machine Webhook Endpoint
Path: `backend/Api/Controllers/AttendanceWebhookController.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Api.Filters;

namespace Api.Controllers;

[ApiController]
[Route("api/webhooks/attendance")]
public class AttendanceWebhookController : ControllerBase
{
    [AllowAnonymous]
    [ApiKeyAuthorize("Integrations:BiometricApiKey")]
    [HttpPost("punch-push")]
    public async Task<IActionResult> ReceivePunchLog([FromBody] BiometricPushLogDto log)
    {
        // Parse device serial, user enrollment number, timestamp
        // await attendanceService.RecordRawPunchAsync(log.DeviceSerial, log.UserCode, log.PunchTime);
        return Ok(new { status = "SUCCESS" });
    }
}
```

---

## 10. Next.js Frontend Integration (`frontend/src/`)

Now that your backend issues JWT tokens packed with `permission` claims, here is how to consume them in your **Next.js App Router** frontend (`frontend/src/`).

### 10.1. Token Payload Type & Helper
Path: `frontend/src/lib/auth.ts`

```typescript
import { jwtDecode } from "jwt-decode";

export interface DecodedToken {
  sub: string;
  email: string;
  fullName: string;
  role: string | string[];
  permission?: string | string[]; // Can be string (if single) or array of strings
  company_id?: string;
  department_id?: string;
  exp: number;
}

export function getUserPermissions(token: string): string[] {
  try {
    const decoded = jwtDecode<DecodedToken>(token);
    if (!decoded.permission) return [];
    return Array.isArray(decoded.permission) ? decoded.permission : [decoded.permission];
  } catch {
    return [];
  }
}

export function hasPermission(token: string, requiredPermission: string): boolean {
  try {
    const decoded = jwtDecode<DecodedToken>(token);
    // Superuser Admin bypass in frontend
    const roles = Array.isArray(decoded.role) ? decoded.role : [decoded.role];
    if (roles.includes("Admin")) return true;

    const perms = getUserPermissions(token);
    return perms.includes(requiredPermission);
  } catch {
    return false;
  }
}
```

---

### 10.2. UI Guard Component (`<HasPermission>`)
Path: `frontend/src/components/HasPermission.tsx`

```tsx
"use client";

import React from "react";
import { useAuth } from "@/lib/useAuth"; // your auth context or Zustand store
import { hasPermission } from "@/lib/auth";

interface Props {
  code: string;
  children: React.ReactNode;
  fallback?: React.ReactNode;
}

export function HasPermission({ code, children, fallback = null }: Props) {
  const { token } = useAuth();

  if (!token || !hasPermission(token, code)) {
    return <>{fallback}</>;
  }

  return <>{children}</>;
}
```

#### Usage in your UI:
Inside any module component (e.g. `frontend/src/modules/leave/` or `frontend/src/modules/payroll/`):

```tsx
import { HasPermission } from "@/components/HasPermission";

export function LeaveApprovalButton({ leaveId }: { leaveId: string }) {
  return (
    <div>
      {/* Visible ONLY to users with 'leave.approve' or 'Admin' role */}
      <HasPermission code="leave.approve">
        <button className="btn btn-primary" onClick={() => approveLeave(leaveId)}>
          Approve Leave
        </button>
      </HasPermission>

      {/* Visible ONLY to users with 'payroll.process' */}
      <HasPermission code="payroll.process">
        <button className="btn btn-warning" onClick={() => runPayroll()}>
          Process Payroll
        </button>
      </HasPermission>
    </div>
  );
}
```

---

## 11. Complete Rollout Checklist for `HrmsSys`

- [ ] **Domain Layer**:
  - [ ] Add `ApplicationUser`, `ApplicationRole`, `Permission`, `RolePermission` in `Domain/Entities/IdentityEntities.cs`.
  - [ ] Add `TemporaryAccessLink` in `Domain/Entities/TemporaryAccessLink.cs` for Option B magic links.
- [ ] **Infrastructure Layer**:
  - [ ] Update `AppDbContext` to inherit from `IdentityDbContext<ApplicationUser, ApplicationRole, string>`.
  - [ ] Add `sys_permissions`, `sys_role_permissions`, and `sys_temporary_links` mappings in `OnModelCreating`.
  - [ ] Create `TokenService.cs` to embed role, permission claims, and company ID.
  - [ ] Create `MagicLinkService.cs` for generating and validating time-bound links.
  - [ ] Add `SeedData.cs` with the HRMS permission catalog and initial roles.
- [ ] **Application Layer**:
  - [ ] Add `PermissionRequirement.cs`, `PermissionPolicyProvider.cs`, and `PermissionAuthorizationHandler.cs` in `Application/Common/Authorization/`.
- [ ] **Api Layer**:
  - [ ] Register Identity, JWT Bearer, and `AddDynamicPermissionAuthorization()` in `Api/Program.cs`.
  - [ ] Add `RequireMagicLinkAttribute.cs` filter in `Api/Filters/`.
  - [ ] Add `ApiKeyAuthorizeAttribute.cs` filter in `Api/Filters/`.
  - [ ] Decorate controllers with `[Authorize]` and actions with `[Authorize(Policy = "...")]`.
- [ ] **Frontend Layer**:
  - [ ] Add `hasPermission` token parser in `frontend/src/lib/auth.ts`.
  - [ ] Wrap sensitive action buttons and links with `<HasPermission code="...">`.
