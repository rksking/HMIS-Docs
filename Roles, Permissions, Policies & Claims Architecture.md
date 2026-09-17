# Enterprise Roles, Permissions, Policies & Claims Architecture: HRMS Implementation Guide

> **Target Audience**: Backend Engineers & Software Architects  
> **Target Framework**: .NET 8 / .NET 9 / .NET 10 (C#)  
> **Source Model**: High-Performance Microservices Authorization Pattern (adapted from HIMS)

---

## 1. Executive Summary & Architecture Overview

The authorization architecture implemented in this codebase is a **hybrid Claims-Based & Dynamic Policy-Driven Access Control (CBAC / PBAC)** engine. It eliminates the traditional maintenance pitfalls of Role-Based Access Control (RBAC) and hardcoded ASP.NET Core policies.

### Core Architectural Pillars

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                                   Identity Server                                      │
│                                                                                        │
│   ┌──────────────┐         ┌──────────────────┐         ┌────────────────────────┐     │
│   │ Application  │ 1     * │   Application    │ *     * │       Permission       │     │
│   │     User     ├─────────┤   Role (Roles)   ├─────────┤   (Granular Catalog)   │     │
│   └──────┬───────┘         └──────────────────┘         └────────────────────────┘     │
│          │                                                                             │
│          │  Login Request                                                              │
│          ▼                                                                             │
│   ┌──────────────────────────────────────────────────────────────────────────────┐     │
│   │ TokenService: Generates stateless JWT                                        │     │
│   │  • sub: User Id                                                              │     │
│   │  • role: ["HR_Manager", "Department_Head"]                                  │     │
│   │  • permission: ["hr.employees.view", "hr.leaves.approve", ...]               │     │
│   │  • location_id / department_id / tenant_id (Scope Context)                   │     │
│   └──────────────────────────────────────┬───────────────────────────────────────┘     │
└──────────────────────────────────────────┼─────────────────────────────────────────────┘
                                           │ Bearer JWT
                                           ▼
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                              Microservices / HRMS API                                  │
│                                                                                        │
│   [Authorize(Policy = "hr.leaves.approve")]                                            │
│   public async Task<IActionResult> ApproveLeave(...)                                   │
│                                                                                        │
│   ┌──────────────────────────────────────────────────────────────────────────────┐     │
│   │ Dynamic Engine (PermissionPolicyProvider):                                  │     │
│   │  1. Any policy name -> Dynamically creates PermissionRequirement("code")     │     │
│   │  2. PermissionAuthorizationHandler:                                          │     │
│   │     - If caller is in Role "Admin" -> SUCCEED (Superuser Bypass)             │     │
│   │     - If caller has Claim "permission" == "hr.leaves.approve" -> SUCCEED     │     │
│   │     - Else -> FAIL (403 Forbidden)                                           │     │
│   └──────────────────────────────────────────────────────────────────────────────┘     │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### Why This Design Excels
1. **Zero Policy Pre-registration**: You do **not** need to register hundreds of `options.AddPolicy("hr.leaves.apply", ...)` lines in `Program.cs`. The dynamic policy provider resolves any policy name string into a permission requirement on the fly.
2. **True Separation of Roles and Permissions**: Users get **Roles**; Roles get **Permissions**. Downstream APIs check **Permissions**, never hardcoded role names (e.g. check for `hr.payroll.process`, not `role == "Accountant"`). Roles can be added or adjusted dynamically in the database without recompiling code.
3. **Admin Superuser Bypass**: The `Admin` role automatically satisfies every permission requirement, eliminating the need to update `Admin` permissions every time a new API endpoint is created.
4. **Stateless Microservices**: Microservices validate incoming JWT tokens and evaluate permission claims locally without querying the database or sending network requests to the Identity Server.

---

## 2. Step 1: Domain Entities & Database Schema

The identity subsystem requires four primary entities: `ApplicationUser`, `ApplicationRole`, `Permission`, and `RolePermission`.

### 2.1. Domain Entities (`Entities.cs`)

```csharp
using Microsoft.AspNetCore.Identity;

namespace HRMS.Identity.Domain;

// 1. Extended Identity User
public class ApplicationUser : IdentityUser
{
    public string FullName { get; set; } = string.Empty;
    public string? Department { get; set; }
    public string? Designation { get; set; }
    public Guid? DepartmentId { get; set; }
    public Guid? LocationId { get; set; }     // Null = Group-wide / Enterprise user
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

// 3. Permission Catalog Entity
public class Permission
{
    public int Id { get; set; }
    public string Code { get; set; } = string.Empty;       // Unique key, e.g. "hr.employees.create"
    public string Name { get; set; } = string.Empty;       // Friendly name, e.g. "Create Employees"
    public string Module { get; set; } = string.Empty;     // Category, e.g. "Employees"
    public string? Description { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    public ICollection<RolePermission> RolePermissions { get; set; } = [];
}

// 4. Join Entity: Role <-> Permission
public class RolePermission
{
    public string RoleId { get; set; } = string.Empty;
    public ApplicationRole Role { get; set; } = null!;

    public int PermissionId { get; set; }
    public Permission Permission { get; set; } = null!;

    public DateTime AssignedAt { get; set; } = DateTime.UtcNow;
}
```

### 2.2. Entity Framework Core DbContext (`AppIdentityDbContext.cs`)

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using HRMS.Identity.Domain;

namespace HRMS.Identity.Infrastructure;

public class AppIdentityDbContext(DbContextOptions<AppIdentityDbContext> options)
    : IdentityDbContext<ApplicationUser, ApplicationRole, string>(options)
{
    public DbSet<Permission> Permissions => Set<Permission>();
    public DbSet<RolePermission> RolePermissions => Set<RolePermission>();

    protected override void OnModelCreating(ModelBuilder builder)
    {
        base.OnModelCreating(builder);

        // Rename standard ASP.NET Identity tables to clean snake_case
        builder.Entity<ApplicationUser>(e => e.ToTable("users"));
        builder.Entity<ApplicationRole>(e => e.ToTable("roles"));
        builder.Entity<IdentityUserRole<string>>(e => e.ToTable("user_roles"));
        builder.Entity<IdentityUserClaim<string>>(e => e.ToTable("user_claims"));
        builder.Entity<IdentityUserLogin<string>>(e => e.ToTable("user_logins"));
        builder.Entity<IdentityUserToken<string>>(e => e.ToTable("user_tokens"));
        builder.Entity<IdentityRoleClaim<string>>(e => e.ToTable("role_claims"));

        // Permissions catalog
        builder.Entity<Permission>(e =>
        {
            e.ToTable("permissions");
            e.HasKey(p => p.Id);
            e.HasIndex(p => p.Code).IsUnique();
            e.Property(p => p.Code).HasMaxLength(100).IsRequired();
            e.Property(p => p.Name).HasMaxLength(150).IsRequired();
            e.Property(p => p.Module).HasMaxLength(100).IsRequired();
        });

        // RolePermissions composite join table
        builder.Entity<RolePermission>(e =>
        {
            e.ToTable("role_permissions");
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
    }
}
```

---

## 3. Step 2: Shared Dynamic Authorization Engine

Place this lightweight engine in a shared library (e.g. `HRMS.SharedKernel` or `HRMS.Common.Authorization`) so every service or API in your HRMS solution can consume it with one line of code.

### 3.1. `PermissionRequirement.cs`

```csharp
using Microsoft.AspNetCore.Authorization;

namespace HRMS.SharedKernel.Authorization;

/// <summary>
/// The permission code itself (e.g. "hr.payroll.process") doubles as both
/// the ASP.NET Core policy name and the authorization requirement.
/// </summary>
public class PermissionRequirement(string permission) : IAuthorizationRequirement
{
    public string Permission { get; } = permission;
}
```

### 3.2. `PermissionPolicyProvider.cs`

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.Extensions.Options;

namespace HRMS.SharedKernel.Authorization;

/// <summary>
/// Dynamically builds an AuthorizationPolicy for any policy name passed to [Authorize(Policy = "...")].
/// Bypasses the requirement to register named policies inside Program.cs.
/// </summary>
public class PermissionPolicyProvider(IOptions<AuthorizationOptions> options) : IAuthorizationPolicyProvider
{
    private readonly DefaultAuthorizationPolicyProvider _fallback = new(options);

    public bool AllowsCachingPolicies => ((IAuthorizationPolicyProvider)_fallback).AllowsCachingPolicies;

    public Task<AuthorizationPolicy> GetDefaultPolicyAsync() => _fallback.GetDefaultPolicyAsync();
    public Task<AuthorizationPolicy?> GetFallbackPolicyAsync() => _fallback.GetFallbackPolicyAsync();

    public Task<AuthorizationPolicy?> GetPolicyAsync(string policyName)
    {
        // Treat any arbitrary policyName as a permission code requirement
        var policy = new AuthorizationPolicyBuilder()
            .RequireAuthenticatedUser()
            .AddRequirements(new PermissionRequirement(policyName))
            .Build();

        return Task.FromResult<AuthorizationPolicy?>(policy);
    }
}
```

### 3.3. `PermissionAuthorizationHandler.cs`

```csharp
using Microsoft.AspNetCore.Authorization;

namespace HRMS.SharedKernel.Authorization;

public class PermissionAuthorizationHandler : AuthorizationHandler<PermissionRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context, 
        PermissionRequirement requirement)
    {
        // 1. Superuser bypass: "Admin" can do everything without individual permission codes
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        // 2. Granular check: Match the token's "permission" claims against the required code
        if (context.User.HasClaim(c => c.Type == "permission" && c.Value == requirement.Permission))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

### 3.4. Dependency Injection Extension (`ServiceCollectionExtensions.cs`)

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.Extensions.DependencyInjection;

namespace HRMS.SharedKernel.Authorization;

public static class PermissionAuthorizationExtensions
{
    /// <summary>
    /// Registers the dynamic policy provider and permission handler.
    /// MUST be called AFTER services.AddAuthorization() in Program.cs.
    /// </summary>
    public static IServiceCollection AddPermissionAuthorization(this IServiceCollection services)
    {
        services.AddSingleton<IAuthorizationPolicyProvider, PermissionPolicyProvider>();
        services.AddScoped<IAuthorizationHandler, PermissionAuthorizationHandler>();
        return services;
    }
}
```

---

## 4. Step 3: Permission Catalog & Seeding Strategy for HRMS

Design your HRMS permission catalog using the hierarchical naming convention:  
`{module}.{resource}.{action}`

### 4.1. HRMS Permission Catalog (`SeedData.cs`)

```csharp
using Microsoft.AspNetCore.Identity;
using Microsoft.EntityFrameworkCore;
using HRMS.Identity.Domain;

namespace HRMS.Identity.Infrastructure;

public static class SeedData
{
    private static readonly (string Module, string Code, string Name)[] Catalog =
    [
        // Employees Module
        ("Employees", "hr.employees.view", "View Employee Records"),
        ("Employees", "hr.employees.create", "Create New Employee"),
        ("Employees", "hr.employees.edit", "Edit Employee Information"),
        ("Employees", "hr.employees.delete", "Delete/Deactivate Employee"),

        // Attendance & Timesheets
        ("Attendance", "hr.attendance.view", "View Attendance Logs"),
        ("Attendance", "hr.attendance.record", "Record/Punch Attendance"),
        ("Attendance", "hr.attendance.manage", "Manage & Override Shifts/Timesheets"),

        // Leave Management
        ("Leaves", "hr.leaves.view", "View Leave Applications"),
        ("Leaves", "hr.leaves.apply", "Apply For Leave (Admin/On Behalf)"),
        ("Leaves", "hr.leaves.approve", "Approve/Reject Leave Applications"),
        ("Leaves", "hr.leaves.manage", "Manage Leave Quotas and Types"),

        // Payroll & Compensation
        ("Payroll", "hr.payroll.view", "View Payroll Register"),
        ("Payroll", "hr.payroll.process", "Process Monthly Payroll & Deductions"),
        ("Payroll", "hr.payroll.pay", "Authorize & Execute Salary Payments"),
        ("Payroll", "hr.payroll.reports", "Generate Statutory & Tax Reports"),

        // Recruitment & Onboarding
        ("Recruitment", "hr.recruitment.view", "View Job Openings and Candidates"),
        ("Recruitment", "hr.recruitment.manage", "Manage Job Postings and Pipelines"),
        ("Recruitment", "hr.recruitment.interview", "Schedule and Score Interviews"),
        ("Recruitment", "hr.recruitment.offer", "Generate and Issue Offer Letters"),

        // Performance & Appraisal
        ("Performance", "hr.performance.view", "View Appraisals & KPIs"),
        ("Performance", "hr.performance.manage", "Initiate Appraisal Cycles"),
        ("Performance", "hr.performance.evaluate", "Submit Manager Evaluations"),

        // Master Data & Organization
        ("Organization", "hr.masters.view", "View Departments, Designations & Grades"),
        ("Organization", "hr.masters.manage", "Manage Departments & Grades"),

        // Administration & Security
        ("Admin", "users.manage", "Manage Users and Accounts"),
        ("Admin", "roles.manage", "Manage Roles and Permission Matrices"),
        ("Admin", "permissions.manage", "Manage System Permissions")
    ];

    // Role Mapping Matrix: Role Name -> Array of granted permission codes (or "*" for full access)
    private static readonly Dictionary<string, (string Description, string[] Perms)> RoleMap = new()
    {
        ["Admin"] = ("System Administrator with Unrestricted Access", ["*"]),
        
        ["HR_Director"] = ("HR Executive / Director", [
            "hr.employees.view", "hr.employees.create", "hr.employees.edit", "hr.employees.delete",
            "hr.attendance.view", "hr.attendance.manage",
            "hr.leaves.view", "hr.leaves.approve", "hr.leaves.manage",
            "hr.payroll.view", "hr.payroll.process", "hr.payroll.pay", "hr.payroll.reports",
            "hr.recruitment.view", "hr.recruitment.manage", "hr.recruitment.interview", "hr.recruitment.offer",
            "hr.performance.view", "hr.performance.manage", "hr.performance.evaluate",
            "hr.masters.view", "hr.masters.manage",
            "users.manage", "roles.manage"
        ]),

        ["HR_Officer"] = ("HR Operations & Recruitment Specialist", [
            "hr.employees.view", "hr.employees.create", "hr.employees.edit",
            "hr.attendance.view", "hr.attendance.manage",
            "hr.leaves.view", "hr.leaves.approve",
            "hr.recruitment.view", "hr.recruitment.manage", "hr.recruitment.interview", "hr.recruitment.offer",
            "hr.performance.view",
            "hr.masters.view"
        ]),

        ["Payroll_Accountant"] = ("Payroll & Compensation Officer", [
            "hr.employees.view",
            "hr.attendance.view",
            "hr.leaves.view",
            "hr.payroll.view", "hr.payroll.process", "hr.payroll.pay", "hr.payroll.reports"
        ]),

        ["Department_Manager"] = ("HOD / Department Lead", [
            "hr.employees.view",
            "hr.attendance.view",
            "hr.leaves.view", "hr.leaves.approve",
            "hr.recruitment.interview",
            "hr.performance.view", "hr.performance.evaluate"
        ]),

        ["Employee"] = ("Standard Staff / Self-Service User", [
            // Standard employees interact via self-service authenticated ownership checks
            "hr.attendance.record",
            "hr.masters.view"
        ])
    };

    public static async Task SeedAsync(
        AppIdentityDbContext db,
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

        // 2. Seed Roles & RolePermissions
        var allPermissions = await db.Permissions.ToListAsync();
        foreach (var (roleName, (desc, perms)) in RoleMap)
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
                ? allPermissions
                : allPermissions.Where(p => perms.Contains(p.Code)).ToList();

            foreach (var perm in granted)
            {
                if (!await db.RolePermissions.AnyAsync(rp => rp.RoleId == role.Id && rp.PermissionId == perm.Id))
                {
                    db.RolePermissions.Add(new RolePermission { RoleId = role.Id, PermissionId = perm.Id });
                }
            }
        }
        await db.SaveChangesAsync();

        // 3. Seed Default Super Admin Account
        const string adminEmail = "admin@hrms.local";
        if (await userManager.FindByEmailAsync(adminEmail) is null)
        {
            var adminUser = new ApplicationUser
            {
                UserName = adminEmail,
                Email = adminEmail,
                FullName = "System Administrator",
                Designation = "Chief Administrator",
                EmailConfirmed = true
            };
            await userManager.CreateAsync(adminUser, "P@ssword123!");
            await userManager.AddToRoleAsync(adminUser, "Admin");
        }
    }
}
```

---

## 5. Step 4: Token Generation & Claims Packaging

When a user logs in, the `TokenService` gathers their identity, roles, and all distinct permissions mapped to their roles, packing them as claims in a signed JWT.

### 5.1. `TokenService.cs`

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Text;
using Microsoft.IdentityModel.Tokens;
using HRMS.Identity.Domain;

namespace HRMS.Identity.Application.Services;

public class TokenService(IConfiguration config)
{
    public (string token, DateTime expires) CreateToken(
        ApplicationUser user, 
        IList<string> roles, 
        IList<string> permissions)
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

        // Add standard Role claims
        claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

        // Add fine-grained custom Permission claims
        claims.AddRange(permissions.Select(p => new Claim("permission", p)));

        // Multi-tenancy / Location / Department claims for automatic scoping
        if (user.LocationId.HasValue)
            claims.Add(new Claim("location_id", user.LocationId.Value.ToString()));

        if (user.DepartmentId.HasValue)
            claims.Add(new Claim("department_id", user.DepartmentId.Value.ToString()));

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

## 6. Step 5: Role & Permission Management APIs

Below are the management endpoints allowing HR/IT administrators to define roles, assign permissions to roles, and bind roles to employee accounts.

### 6.1. DTOs

```csharp
namespace HRMS.Identity.Application.DTOs;

public record CreateRoleRequest(string Name, string? Description, List<string> PermissionCodes);
public record UpdateRolePermissionsRequest(string RoleName, List<string> PermissionCodes);
public record AssignRoleRequest(string Email, string Role);
public record RoleDto(string Id, string Name, string? Description, bool IsSystemRole, List<string> Permissions);
public record PermissionDto(int Id, string Code, string Name, string Module);
```

### 6.2. Roles & Permissions Application Service (`IdentityAppService.cs`)

```csharp
public async Task<ApiResponse<RoleDto>> CreateRoleAsync(CreateRoleRequest req)
{
    if (await roleManager.RoleExistsAsync(req.Name))
        throw new ConflictException($"Role {req.Name} already exists.");

    var role = new ApplicationRole(req.Name) { Description = req.Description };
    await roleManager.CreateAsync(role);

    await SetRolePermissionsAsync(role.Id, req.PermissionCodes);

    return ApiResponse<RoleDto>.Ok(
        new RoleDto(role.Id, role.Name!, role.Description, role.IsSystemRole, req.PermissionCodes),
        "Role created.");
}

public async Task<ApiResponse<bool>> UpdateRolePermissionsAsync(UpdateRolePermissionsRequest req)
{
    var role = await roleManager.FindByNameAsync(req.RoleName)
        ?? throw new NotFoundException("Role", req.RoleName);

    var existing = db.RolePermissions.Where(rp => rp.RoleId == role.Id);
    db.RolePermissions.RemoveRange(existing);
    await db.SaveChangesAsync();

    await SetRolePermissionsAsync(role.Id, req.PermissionCodes);
    return ApiResponse<bool>.Ok(true, $"Permissions updated for role {req.RoleName}.");
}

public async Task<ApiResponse<bool>> AssignRoleAsync(AssignRoleRequest req)
{
    var user = await userManager.FindByEmailAsync(req.Email)
        ?? throw new NotFoundException("User", req.Email);

    if (!await roleManager.RoleExistsAsync(req.Role))
        throw new NotFoundException("Role", req.Role);

    await userManager.AddToRoleAsync(user, req.Role);
    return ApiResponse<bool>.Ok(true, $"Role {req.Role} assigned to {req.Email}.");
}

private async Task<List<string>> GetPermissionsForRolesAsync(IList<string> roleNames)
{
    var roleIds = await roleManager.Roles
        .Where(r => roleNames.Contains(r.Name!))
        .Select(r => r.Id)
        .ToListAsync();

    return await db.RolePermissions
        .Where(rp => roleIds.Contains(rp.RoleId))
        .Include(rp => rp.Permission)
        .Select(rp => rp.Permission.Code)
        .Distinct()
        .ToListAsync();
}
```

---

## 7. Step 6: Controller & Action Level Implementation

### 7.1. Microservice Configuration (`Program.cs`)

> [!IMPORTANT]
> **DI Registration Order Matters**: Call `builder.Services.AddPermissionAuthorization()` **immediately after** `builder.Services.AddAuthorization(...)`. In .NET DI, when resolving a singleton interface like `IAuthorizationPolicyProvider`, the last registered service wins.

```csharp
// 1. JWT Authentication
var jwt = builder.Configuration.GetSection("Jwt");
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(JwtBearerDefaults.AuthenticationScheme, o =>
    {
        o.RequireHttpsMetadata = false;
        o.TokenValidationParameters = new TokenValidationParameters
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

// 2. Global Fallback Policy (Default Deny - Requires authentication everywhere)
builder.Services.AddAuthorization(o =>
    o.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build());

// 3. Dynamic Permission Authorization Engine
builder.Services.AddPermissionAuthorization();
```

### 7.2. Controller Decoration Pattern

```csharp
using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace HRMS.HRService.API.Controllers;

[ApiController]
[Route("api/hr")]
[Authorize] // Enforces authentication at the controller class level
public class HRController(IHRAppService svc) : ControllerBase
{
    private string CurrentUserId => User.FindFirstValue(ClaimTypes.NameIdentifier) ?? "system";

    // ── 1. Fine-Grained Policy Protection (Action Level) ──
    [Authorize(Policy = "hr.employees.create")]
    [HttpPost("employees")]
    public async Task<IActionResult> CreateEmployee([FromBody] CreateEmployeeRequest req)
        => Ok(await svc.CreateEmployeeAsync(req, CurrentUserId));

    [Authorize(Policy = "hr.employees.view")]
    [HttpGet("employees")]
    public async Task<IActionResult> GetEmployees([FromQuery] EmployeeFilter filter)
        => Ok(await svc.GetEmployeesAsync(filter));

    [Authorize(Policy = "hr.leaves.approve")]
    [HttpPut("leaves/{id:guid}/approve")]
    public async Task<IActionResult> ApproveLeave(Guid id)
        => Ok(await svc.ApproveLeaveAsync(id, CurrentUserId));

    // ── 2. Self-Service vs Administrative Permissions ──
    // Scenario: An employee should be able to apply for their own leave without needing
    // administrative "hr.leaves.manage" permissions.
    [HttpPost("leaves/apply-my-leave")]
    public async Task<IActionResult> ApplyMyLeave([FromBody] ApplyLeaveRequest req)
    {
        // Enforce ownership: caller can only apply for themselves
        var employeeId = await svc.GetEmployeeIdByUserIdAsync(CurrentUserId);
        req.EmployeeId = employeeId;
        return Ok(await svc.ApplyLeaveAsync(req));
    }

    // Scenario: Viewing a payslip
    // Administrative check (hr.payroll.view) OR Ownership check (caller's own payslip)
    [HttpGet("payroll/payslips/{id:guid}")]
    public async Task<IActionResult> GetPayslip(Guid id)
    {
        var payslip = await svc.GetPayslipAsync(id);
        var isHrPayrollAdmin = User.HasClaim("permission", "hr.payroll.view") || User.IsInRole("Admin");

        var callerEmployeeId = await svc.GetEmployeeIdByUserIdAsync(CurrentUserId);
        if (!isHrPayrollAdmin && payslip.EmployeeId != callerEmployeeId)
        {
            return Forbid(); // 403 Forbidden: Caller cannot view other employees' payslips
        }

        return Ok(payslip);
    }
}
```

---

## 8. Step 7: Callbacks, Webhooks & Public Form Filing

In an HRMS, certain endpoints cannot rely on standard user login tokens:
1. **Public Job Applicant Form** (Career site submission).
2. **Pre-onboarding Candidate Form** (New hire filling bank details/documents before receiving an employee account).
3. **Biometric Punch Machine Webhook / Push Callback** (Hardware clock syncing punch data into attendance).
4. **Third-Party Payroll / Tax Webhook** (Bank callback for salary disbursement confirmation).

Here are the 3 production-grade options for handling these scenarios.

```
┌────────────────────────────────────────────────────────────────────────────────────────┐
│                        Public & Inbound Integration Options                            │
├────────────────────────────────┬───────────────────────┬───────────────────────────────┤
│ Use Case                       │ Mechanism             │ Security Control              │
├────────────────────────────────┼───────────────────────┼───────────────────────────────┤
│ Job Application Form           │ [AllowAnonymous]      │ Captcha + Rate Limiting       │
│ Pre-Onboarding Candidate Form  │ Magic Invitation Link │ HMAC-signed Temporary Token   │
│ Biometric Punch Machine Push   │ Webhook / API Key     │ HMAC-SHA256 Header Validation │
└────────────────────────────────┴───────────────────────┴───────────────────────────────┘
```

---

### Option A: Completely Public Form Filing (`[AllowAnonymous]`)

Use for public job portals, careers pages, or anonymous whistleblowing forms.

```csharp
[ApiController]
[Route("api/public/careers")]
public class PublicCareersController(ICareersAppService svc) : ControllerBase
{
    /// <summary>
    /// Overrides the global FallbackPolicy and allows unauthenticated submissions.
    /// Protected with Rate Limiting and Captcha validation.
    /// </summary>
    [AllowAnonymous]
    [HttpPost("apply")]
    public async Task<IActionResult> SubmitJobApplication([FromBody] PublicJobApplicationRequest req)
    {
        // 1. Validate CAPTCHA token (Cloudflare Turnstile or Google reCAPTCHA)
        if (!await svc.VerifyCaptchaTokenAsync(req.CaptchaToken))
            return BadRequest("Invalid CAPTCHA verification.");

        // 2. Persist application
        var result = await svc.SubmitApplicationAsync(req);
        return Ok(result);
    }
}
```

---

### Option B: Secure Temporary Token for Candidate Pre-Onboarding

When a candidate is hired, send them a private link:  
`https://hrms.company.com/onboarding?token=eyJhbGciOi...`  
They can fill in their personal details, emergency contacts, and upload identification without having an active employee account.

#### 1. Generation Logic in HR Service:
```csharp
public string GenerateOnboardingToken(Guid candidateId, string email)
{
    var tokenHandler = new JwtSecurityTokenHandler();
    var key = Encoding.UTF8.GetBytes(_config["Jwt:PreOnboardingSecret"]!);

    var tokenDescriptor = new SecurityTokenDescriptor
    {
        Subject = new ClaimsIdentity([
            new Claim("candidate_id", candidateId.ToString()),
            new Claim("email", email),
            new Claim("scope", "pre_onboarding_form")
        ]),
        Expires = DateTime.UtcNow.AddDays(7), // Link valid for 7 days
        SigningCredentials = new SigningCredentials(new SymmetricSecurityKey(key), SecurityAlgorithms.HmacSha256)
    };

    var token = tokenHandler.CreateToken(tokenDescriptor);
    return tokenHandler.WriteToken(token);
}
```

#### 2. Candidate Controller Consuming the Token:
```csharp
[ApiController]
[Route("api/public/onboarding")]
public class CandidateOnboardingController(IOnboardingAppService svc, IConfiguration config) : ControllerBase
{
    [AllowAnonymous]
    [HttpPost("submit")]
    public async Task<IActionResult> SubmitOnboardingForm(
        [FromHeader(Name = "X-Onboarding-Token")] string token,
        [FromBody] CandidateOnboardingDto dto)
    {
        // 1. Validate the dedicated onboarding token
        var principal = ValidateTemporaryToken(token, config["Jwt:PreOnboardingSecret"]!);
        if (principal is null)
            return Unauthorized("Invalid or expired onboarding invitation link.");

        var candidateId = Guid.Parse(principal.FindFirst("candidate_id")!.Value);

        // 2. Save submitted data against the candidate record
        await svc.SaveCandidateDetailsAsync(candidateId, dto);
        return Ok(new { Message = "Onboarding information successfully submitted." });
    }

    private static ClaimsPrincipal? ValidateTemporaryToken(string token, string secret)
    {
        var tokenHandler = new JwtSecurityTokenHandler();
        var key = Encoding.UTF8.GetBytes(secret);

        try
        {
            return tokenHandler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = new SymmetricSecurityKey(key),
                ValidateIssuer = false,
                ValidateAudience = false,
                ClockSkew = TimeSpan.FromMinutes(2)
            }, out _);
        }
        catch
        {
            return null;
        }
    }
}
```

---

### Option C: Inbound Push Callbacks & Hardware Webhooks (Biometric Punch Machines)

Biometric devices (e.g. ZKTeco, Hikvision, Essl) push real-time attendance logs to your API. Since these devices cannot perform interactive JWT logins, secure them using a **Pre-Shared API Key** or **HMAC-SHA256 Signature Verification**.

#### 1. Custom Action Filter for Machine Authentication (`ApiKeyAuthorizeAttribute.cs`):
```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;

namespace HRMS.HRService.API.Security;

[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class)]
public class ApiKeyAuthorizeAttribute(string configKey = "Integrations:BiometricApiKey") : Attribute, IAsyncActionFilter
{
    private const string ApiKeyHeaderName = "X-API-KEY";

    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var configuration = context.HttpContext.RequestServices.GetRequiredService<IConfiguration>();
        var expectedKey = configuration[configKey];

        if (!context.HttpContext.Request.Headers.TryGetValue(ApiKeyHeaderName, out var extractedKey) ||
            string.IsNullOrWhiteSpace(extractedKey) ||
            !CryptographicEquals(expectedKey!, extractedKey!))
        {
            context.Result = new UnauthorizedObjectResult(new { message = "Invalid or missing Machine API Key." });
            return;
        }

        await next();
    }

    // Time-constant comparison to protect against timing attacks
    private static bool CryptographicEquals(string a, string b)
    {
        var aBytes = Encoding.UTF8.GetBytes(a);
        var bBytes = Encoding.UTF8.GetBytes(b);
        return System.Security.Cryptography.CryptographicOperations.FixedTimeEquals(aBytes, bBytes);
    }
}
```

#### 2. Punch Machine Webhook Endpoint:
```csharp
[ApiController]
[Route("api/webhooks/attendance")]
public class BiometricWebhookController(IAttendanceAppService svc) : ControllerBase
{
    /// <summary>
    /// Receives real-time push events from biometric hardware devices.
    /// Bypass JWT ([AllowAnonymous]) while strictly validating X-API-KEY.
    /// </summary>
    [AllowAnonymous]
    [ApiKeyAuthorize("Integrations:BiometricApiKey")]
    [HttpPost("punch-log")]
    public async Task<IActionResult> ReceiveBiometricPunch([FromBody] BiometricPunchLogRequest log)
    {
        await svc.ProcessRawPunchAsync(log.DeviceSerialNumber, log.EnrollmentNumber, log.PunchTime);
        return Ok(new { status = "ACK" });
    }
}
```

---

## 9. Complete Implementation Checklist for HRMS

Use this checklist when rolling out this security architecture into your HRMS:

- [ ] **1. Shared Kernel Setup**:
  - [ ] Add `PermissionRequirement.cs`
  - [ ] Add `PermissionPolicyProvider.cs`
  - [ ] Add `PermissionAuthorizationHandler.cs`
  - [ ] Add `PermissionAuthorizationExtensions.cs`
- [ ] **2. Identity Database Setup**:
  - [ ] Configure `ApplicationUser`, `ApplicationRole`, `Permission`, `RolePermission`
  - [ ] Configure `AppIdentityDbContext` table mappings and composite keys
  - [ ] Run EF Core Migrations: `dotnet ef migrations add InitialIdentitySetup`
- [ ] **3. Seed Catalog & Baseline Roles**:
  - [ ] Define catalog array for HRMS modules (`Employees`, `Attendance`, `Leaves`, `Payroll`, `Recruitment`, `Performance`)
  - [ ] Define role mappings for `Admin`, `HR_Director`, `Department_Manager`, `Employee`, etc.
  - [ ] Execute `SeedData.SeedAsync(...)` during application startup
- [ ] **4. Token Issuance**:
  - [ ] Ensure `TokenService` maps roles and all associated permissions into `"permission"` claims
  - [ ] Ensure tenant/branch claims (e.g. `location_id`, `department_id`) are included
- [ ] **5. Microservice / API Program.cs**:
  - [ ] Add `AddAuthentication(...)` with JWT Bearer
  - [ ] Add `AddAuthorization(...)` with fallback policy requiring authenticated users
  - [ ] Add `AddPermissionAuthorization()` **after** `AddAuthorization()`
- [ ] **6. Controllers**:
  - [ ] Decorate controllers with `[Authorize]`
  - [ ] Decorate action methods with granular policies: `[Authorize(Policy = "hr.payroll.process")]`
  - [ ] Handle self-service actions (e.g. applying for own leave, viewing own payslip) via caller ownership checks
- [ ] **7. Callbacks & Public Forms**:
  - [ ] Add `[AllowAnonymous]` + Captcha for public applicant submissions
  - [ ] Use signed temporary JWTs for candidate pre-onboarding links
  - [ ] Use `[ApiKeyAuthorize]` or HMAC validation for biometric hardware attendance pushes

---

## 10. Common Pitfalls & Security Best Practices

1. **JWT Claim Type Mapping Issue**:
   By default, ASP.NET Core remaps short claim names (`sub`, `role`) to lengthy XML schema URIs. If you observe `User.IsInRole(...)` or `User.FindAll("permission")` failing, disable default inbound claim mapping at startup:
   ```csharp
   JwtSecurityTokenHandler.DefaultInboundClaimTypeMap.Clear();
   ```
2. **Token Size Management**:
   If an enterprise has 500+ granular permissions, packing all permissions into a JWT might inflate token headers past standard 8KB limits. In our design, permissions are mapped to roles, and users typically have 20–60 permissions (~1.5KB). If your catalog grows significantly:
   - Use short permission codes (e.g. `hr.emp.c`, `hr.pay.p`).
   - Or implement token caching where token carries only a Session ID / Role ID, and permissions are cached in Redis.
3. **Admin Role Consistency**:
   In `PermissionAuthorizationHandler`, ensure the role check matches your administrator role name:
   ```csharp
   if (context.User.IsInRole("Admin")) { context.Succeed(requirement); return Task.CompletedTask; }
   ```
4. **Immediate Revocation (Handling Stale Tokens)**:
   Because JWT tokens are stateless, permission changes only take effect when a user logs in or refreshes their token. For instant privilege revocation:
   - Set a low access token lifetime (e.g., 15–30 minutes) paired with a Refresh Token.
   - For high-security actions (e.g. executing payroll payouts), perform a real-time check against the database or a Redis blacklist.
