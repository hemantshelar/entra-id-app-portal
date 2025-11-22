# Entra ID App Registration Portal - Technical Requirements Document

## 1. Project Overview

### 1.1 Purpose

Build a modern web portal to manage and monitor Azure Entra ID (formerly Azure AD) application registrations with filtering, monitoring, and management capabilities.

### 1.2 Goals

- Provide a secure, authenticated web portal for authorized users only
- **Use Managed Identity for all Azure resource access (zero secrets in production)**
- **Enable easy local development** - same code works locally and in Azure
- Provide a user-friendly interface to view and manage Entra ID app registrations
- Enable proactive monitoring of expiring secrets and certificates
- Reduce Microsoft Graph API calls through intelligent caching
- Implement group-based authorization using Entra ID security groups
- Deploy to Azure cloud with scalability and security best practices

### 1.3 Security-First Design Principle

**🔐 ZERO SECRETS ARCHITECTURE**: This application follows a zero-secrets architecture using Azure Managed Identity and DefaultAzureCredential:

- ✅ **NO** client secrets stored in configuration files
- ✅ **NO** connection strings with credentials
- ✅ **NO** API keys in code or environment variables
- ✅ All Azure resources accessed via **Managed Identity**
- ✅ All credentials managed automatically by Azure
- ✅ Developers use their own credentials locally (Azure CLI, VS Code)
- ✅ Production uses **User-Assigned Managed Identity** (reusable across resources)

**Why User-Assigned Managed Identity?**
- ✅ Independent lifecycle (not tied to App Service)
- ✅ Can be shared across multiple Azure resources (Web App, Functions, VMs)
- ✅ Created and configured before deployment
- ✅ Easier to manage permissions centrally
- ✅ Better for multi-resource deployments (Aspire scenarios)

**Why Easy Local Development Matters:**
- ✅ **Same configuration code** works locally and in Azure (no conditional logic!)
- ✅ **Same permissions** - Development MI mirrors production MI permissions
- ✅ **F5 debugging** - No admin permissions needed for individual developers
- ✅ **User Secrets** (`secrets.json`) keeps configuration local and out of git
- ✅ **DefaultAzureCredential** automatically picks right credential (Dev MI or Azure CLI)
- ✅ **No environment detection code** needed - it just works!
- ✅ **Fast development cycle** - no deployment needed to test

**Development Managed Identity Approach:**
Instead of requiring developers to have admin permissions (like `Application.Read.All`), we create a **Development User-Assigned MI** that:
- Has the same Graph API permissions as production MI
- Is shared by all developers
- Eliminates need for individual admin permissions
- Provides consistent testing environment
- Works seamlessly with DefaultAzureCredential

### 1.4 Target Audience

- IT Administrators
- DevOps Engineers
- Security Teams managing Azure Entra ID

---

## 2. Technology Stack

### 2.1 Framework & Runtime

- **.NET Version**: .NET 10.0 (latest)
- **Aspire Framework**: Latest version for cloud-native application development
- **Language**: C# 12+

### 2.2 Frontend

- **UI Framework**: Blazor Server or Blazor Web App (Interactive Server)
- **CSS Framework**: Bootstrap 5+ (responsive design)
- **JavaScript**: Minimal, only when necessary
- **State Management**: Blazor component state with proper loading indicators

### 2.3 Backend

- **Web API**: ASP.NET Core Minimal API or Web API
- **Authentication**: Microsoft Identity Platform (Entra ID)
- **Graph API Client**: Microsoft.Graph SDK (latest)
- **Azure Authentication**: Azure.Identity SDK with DefaultAzureCredential
- **Managed Identity**: System-assigned or User-assigned Managed Identity for Azure resources
- **Caching**: IMemoryCache or IDistributedCache (Redis for production)

### 2.4 Architecture Patterns

- **Dependency Injection (DI)**: Built-in ASP.NET Core DI container
- **Repository Pattern**: For data access abstraction
- **Service Layer Pattern**: Business logic separation
- **CQRS Pattern**: For command/query separation (if complexity requires)
- **Testing**: BDD with SpecFlow or xUnit with FluentAssertions

### 2.5 Cloud & DevOps

- **Hosting**: Azure App Service or Azure Container Apps
- **Identity**: **User-Assigned Managed Identity** (reusable across resources)
- **Configuration**: Azure Key Vault accessed via Managed Identity (no secrets in code)
- **Monitoring**: Application Insights (connected via Managed Identity)
- **Resource Access**: DefaultAzureCredential for all Azure services
- **CI/CD**: GitHub Actions or Azure DevOps

---

## 3. Functional Requirements

### 3.1 Application Registration Management

#### FR-1: View All App Registrations

**Priority**: High  
**Description**: Display all Entra ID app registrations in a paginated, sortable table/grid.

**Acceptance Criteria**:

- Display key properties: Display Name, Application ID, Object ID, Created Date, Owner
- Support pagination (50, 100, 200 items per page)
- Support sorting by any column (ascending/descending)
- Display total count of applications
- Show loading spinner during data fetch
- Handle and display errors gracefully

**Data Fields to Display**:

- Display Name
- Application (Client) ID
- Object ID
- Created Date
- Publisher Domain
- Owners (first owner or count)
- Status (Active/Inactive)
- Secret/Certificate Expiry Status (visual indicator)

---

#### FR-2: Filter App Registrations

**Priority**: High  
**Description**: Enable users to filter applications by various criteria.

**Filter Options**:

1. **Search by Name**: Free-text search (case-insensitive, partial match)
2. **Expiring Secrets**: Apps with secrets expiring within configurable days (default: 30, 60, 90 days)
3. **Expired Secrets**: Apps with already expired secrets
4. **No Secrets**: Apps without any secrets configured
5. **Certificate Expiry**: Similar to secrets (expiring/expired/none)
6. **Created Date Range**: Filter by creation date (from-to)
7. **Owner Filter**: Filter by specific owner email/UPN
8. **Publisher Domain**: Filter by publisher domain

**Acceptance Criteria**:

- All filters work on cached/in-memory data (no API calls)
- Multiple filters can be applied simultaneously (AND logic)
- Clear filter button resets all filters
- Filter state is visually indicated
- Filter results update in real-time
- Display count of filtered results

---

#### FR-3: Delete App Registration

**Priority**: High  
**Description**: Allow authorized users to delete app registrations.

**Acceptance Criteria**:

- Delete button/action available per app row
- Confirmation modal before deletion (show app name and ID)
- Require explicit reason/comment for deletion (audit trail)
- Show success/failure notification
- Refresh cache after successful deletion
- Disable delete button during operation
- Only users with appropriate permissions can delete
- Implement soft-delete option (if audit requirements exist)

**Security**:

- Require additional authentication for delete operations
- Log all deletion attempts (successful and failed)
- Implement role-based access control (RBAC)

---

#### FR-4: Expiring Secrets Dashboard

**Priority**: High  
**Description**: Dedicated view for secrets/certificates nearing expiration.

**Acceptance Criteria**:

- Visual dashboard with color-coded categories:
  - Red: Expired (0 days or less)
  - Orange: Critical (< 7 days)
  - Yellow: Warning (7-30 days)
  - Green: Healthy (> 30 days)
- Group by expiry timeframe
- Show secret details: Description, Key ID, Expiry Date, Days Remaining
- Export capability (CSV/Excel)
- Email notification option (future enhancement)

---

#### FR-5: Data Refresh Mechanism

**Priority**: High  
**Description**: Load data from Microsoft Graph API and provide manual refresh.

**Acceptance Criteria**:

- **Initial Load**: Fetch all app registrations on application start or first user access
- **Batch Loading**: Implement pagination for Graph API calls (batch size: 100-999)
- **Progress Indicator**: Show progress bar during batch loading (e.g., "Loading 300/1000 apps...")
- **Manual Refresh Button**:
  - Clear cache and reload all data
  - Disable button during refresh
  - Show last refresh timestamp
  - Display progress during refresh
- **Auto-refresh**: Optional configurable auto-refresh (default: disabled, option for every 15/30/60 minutes)
- **Error Handling**: Retry logic with exponential backoff for failed API calls

---

#### FR-6: App Registration Details View

**Priority**: Medium  
**Description**: Click on an app to view detailed information.

**Details to Display**:

- All basic properties
- All secrets with expiry dates
- All certificates with expiry dates
- API permissions granted
- Redirect URIs
- Owners list
- Service Principal details (if exists)

**Acceptance Criteria**:

- Open in modal or side panel
- Allow editing of basic properties (stretch goal)
- Copy to clipboard functionality for IDs
- Deep link to Azure Portal for full management

---

#### FR-7: Export Functionality

**Priority**: Medium  
**Description**: Export filtered data to various formats.

**Export Formats**:

- CSV
- Excel (XLSX)
- JSON

**Acceptance Criteria**:

- Export current filtered view
- Include all displayed columns
- Export button disabled when no data
- Show download progress for large datasets

---

#### FR-8: Search with Advanced Options

**Priority**: Medium  
**Description**: Advanced search capabilities.

**Features**:

- Search across multiple fields (name, app ID, object ID)
- Regex support (optional)
- Search history (last 5 searches)
- Saved search filters (user preferences)

---

### 3.2 Additional Features (Nice-to-Have)

#### FR-9: Bulk Operations

- Bulk delete (with confirmation)
- Bulk export

#### FR-10: Analytics Dashboard

- Total apps count
- Apps created this month
- Apps with no owners
- Apps not used (no sign-ins)

#### FR-11: Audit Log

- View history of operations performed
- Filter by user, action, date

---

## 4. Non-Functional Requirements

### 4.1 Performance

- **Initial Load Time**: < 5 seconds for up to 1000 apps
- **Filter Response Time**: < 500ms for any filter operation on cached data
- **API Batch Size**: 100-999 apps per Graph API request
- **Cache Duration**: Configurable (default: until manual refresh)
- **Concurrent Users**: Support at least 50 concurrent users

### 4.2 Scalability

- Horizontal scaling capability (stateless design)
- Support for 10,000+ app registrations
- Efficient memory usage for caching (monitor and optimize)

### 4.3 Security

- **Authentication**: Azure Entra ID authentication (MSAL)
- **Authorization**: Role-based access control (Reader, Contributor, Admin)
- **API Permissions**: Minimum required Graph API permissions:
  - `Application.Read.All` (or `Application.ReadWrite.All` for delete)
  - `Directory.Read.All`
- **Secrets Management**: Never log or expose client secrets
- **Audit Logging**: Log all sensitive operations (delete, update)
- **HTTPS Only**: Enforce HTTPS in production
- **CORS**: Restrict to known origins
- **CSP**: Content Security Policy headers

### 4.4 Reliability

- **Error Handling**: Graceful error handling with user-friendly messages
- **Retry Logic**: Implement for transient Graph API failures
- **Circuit Breaker**: Prevent cascading failures
- **Logging**: Structured logging with Serilog or Microsoft.Extensions.Logging
- **Health Checks**: Implement health check endpoints for monitoring

### 4.5 Usability

- **Responsive Design**: Mobile, tablet, and desktop support
- **Accessibility**: WCAG 2.1 AA compliance
- **Loading States**: Clear indicators for all async operations
- **Error Messages**: Specific, actionable error messages
- **Consistent UI**: Follow Microsoft Fluent Design or Bootstrap patterns
- **Help/Documentation**: Inline help tooltips and user guide

### 4.6 Maintainability

- **Code Quality**: Follow C# coding conventions and StyleCop rules
- **Unit Test Coverage**: Minimum 80% code coverage
- **Integration Tests**: Cover critical user flows
- **Documentation**: XML comments for all public APIs
- **Clean Architecture**: Separate concerns (UI, Business Logic, Data Access)

---

## 5. Technical Design

### 5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Aspire App Host                          │
│  (Orchestration, Service Discovery, Telemetry)              │
└─────────────────────────────────────────────────────────────┘
                            │
            ┌───────────────┴───────────────┐
            │                               │
┌───────────▼──────────┐         ┌─────────▼──────────┐
│   Blazor Web App     │         │   API Service      │
│   (Frontend)         │◄────────┤   (Backend)        │
│                      │  HTTP   │                    │
│  - UI Components     │         │  - Graph Client    │
│  - State Management  │         │  - Cache Manager   │
│  - User Interaction  │         │  - Business Logic  │
└──────────────────────┘         └─────────┬──────────┘
                                           │
                                           │ Graph API
                                           │
                                 ┌─────────▼──────────┐
                                 │  Microsoft Graph   │
                                 │  (Entra ID)        │
                                 └────────────────────┘
```

### 5.2 Project Structure

```
entra-id-app-portal/
├── entra-id-app-portal.AppHost/          # Aspire orchestration
├── entra-id-app-portal.ServiceDefaults/  # Shared configurations
├── entra-id-app-portal.Web/              # Blazor frontend
│   ├── Components/
│   │   ├── Pages/
│   │   │   ├── AppRegistrations.razor    # Main listing page
│   │   │   ├── ExpiringSecrets.razor     # Dashboard
│   │   │   ├── AppDetails.razor          # Details modal
│   │   └── Shared/
│   │       ├── FilterPanel.razor
│   │       ├── AppRegistrationGrid.razor
│   │       ├── LoadingSpinner.razor
│   │       └── ConfirmDialog.razor
│   ├── Services/
│   │   └── IAppRegistrationService.cs    # HTTP client wrapper
│   └── Models/
│       └── ViewModels.cs
├── entra-id-app-portal.ApiService/       # Backend API
│   ├── Controllers/
│   │   └── AppRegistrationsController.cs
│   ├── Services/
│   │   ├── IGraphService.cs
│   │   ├── GraphService.cs               # Graph API calls
│   │   ├── ICacheService.cs
│   │   ├── CacheService.cs               # Cache management
│   │   ├── IUserAuthorizationService.cs  # Group-based authorization
│   │   └── UserAuthorizationService.cs   # Check user group membership
│   ├── Models/
│   │   ├── AppRegistrationDto.cs
│   │   └── FilterOptions.cs
│   └── Repositories/
│       ├── IAppRegistrationRepository.cs
│       └── AppRegistrationRepository.cs
├── entra-id-app-portal.Core/             # Shared models/interfaces
│   ├── Models/
│   ├── Interfaces/
│   └── Exceptions/
└── entra-id-app-portal.Tests/            # Test projects
    ├── Unit/
    ├── Integration/
    └── BDD/
```

### 5.3 Data Models

#### AppRegistrationDto

```csharp
public class AppRegistrationDto
{
    public string Id { get; set; }                      // Object ID
    public string AppId { get; set; }                   // Application ID
    public string DisplayName { get; set; }
    public DateTime CreatedDateTime { get; set; }
    public string PublisherDomain { get; set; }
    public List<string> Owners { get; set; }
    public List<SecretDto> Secrets { get; set; }
    public List<CertificateDto> Certificates { get; set; }
    public ExpiryStatus Status { get; set; }
    public int? DaysUntilExpiry { get; set; }
}

public class SecretDto
{
    public string KeyId { get; set; }
    public string DisplayName { get; set; }
    public DateTime? EndDateTime { get; set; }
    public int? DaysUntilExpiry { get; set; }
    public bool IsExpired { get; set; }
}

public class CertificateDto
{
    public string KeyId { get; set; }
    public string DisplayName { get; set; }
    public DateTime? EndDateTime { get; set; }
    public int? DaysUntilExpiry { get; set; }
    public bool IsExpired { get; set; }
    public string Thumbprint { get; set; }
}

public enum ExpiryStatus
{
    Expired,
    Critical,      // < 7 days
    Warning,       // 7-30 days
    Healthy,       // > 30 days
    NoSecrets
}

public class FilterOptions
{
    public string SearchText { get; set; }
    public int? ExpiringInDays { get; set; }
    public bool? ShowExpired { get; set; }
    public bool? ShowNoSecrets { get; set; }
    public DateTime? CreatedAfter { get; set; }
    public DateTime? CreatedBefore { get; set; }
    public string OwnerFilter { get; set; }
    public string PublisherDomain { get; set; }
    public ExpiryStatus? StatusFilter { get; set; }
}
```

### 5.4 API Endpoints

#### Base URL: `/api/appregistrations`

| Method | Endpoint    | Description                             | Request Body    | Response                   |
| ------ | ----------- | --------------------------------------- | --------------- | -------------------------- |
| GET    | `/`         | Get all cached app registrations        | -               | `List<AppRegistrationDto>` |
| GET    | `/{id}`     | Get single app registration details     | -               | `AppRegistrationDto`       |
| POST   | `/refresh`  | Refresh cache from Graph API            | -               | `RefreshStatus`            |
| POST   | `/filter`   | Apply filters (client-side alternative) | `FilterOptions` | `List<AppRegistrationDto>` |
| DELETE | `/{id}`     | Delete app registration                 | `DeleteRequest` | `DeleteResponse`           |
| GET    | `/expiring` | Get apps with expiring secrets          | `?days=30`      | `List<AppRegistrationDto>` |
| GET    | `/export`   | Export data                             | `?format=csv`   | File download              |
| GET    | `/status`   | Get cache status                        | -               | `CacheStatus`              |

#### Response Models

```csharp
public class RefreshStatus
{
    public int TotalApps { get; set; }
    public int ProcessedApps { get; set; }
    public bool IsComplete { get; set; }
    public DateTime? LastRefreshTime { get; set; }
    public string ErrorMessage { get; set; }
}

public class CacheStatus
{
    public int CachedAppsCount { get; set; }
    public DateTime? LastRefreshTime { get; set; }
    public bool IsRefreshing { get; set; }
}

public class DeleteRequest
{
    public string Reason { get; set; }
    public string PerformedBy { get; set; }
}

public class DeleteResponse
{
    public bool Success { get; set; }
    public string Message { get; set; }
}
```

---

## 6. Configuration Management

### 6.1 Configuration Hierarchy (Priority Order)

**IMPORTANT**: The same configuration structure works both locally and in Azure. Only the source changes!

**Local Development (Priority Order - Highest to Lowest):**
1. **User Secrets** (`secrets.json`) - Highest priority for sensitive local values
2. **Local Settings** (`appsettings.Development.json` or `local.settings.json`) - Local overrides
3. **Base Settings** (`appsettings.json`) - Default values
4. **Environment Variables** - Final fallback

**Azure Production (Priority Order - Highest to Lowest):**
1. **Azure App Configuration** (optional) - Dynamic configuration
2. **Azure Key Vault** - Secrets loaded via Managed Identity
3. **App Service Configuration** (App Settings) - Environment-specific values
4. **Base Settings** (`appsettings.json`) - Default values
5. **Environment Variables** - Final fallback

**Key Benefits**:
- ✅ Same JSON structure works everywhere
- ✅ Developers never touch production secrets
- ✅ DefaultAzureCredential automatically selects correct auth method
- ✅ No code changes between local and cloud
- ✅ Easy to debug locally with Azure CLI login

### 6.2 Required Configuration Keys

```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "<tenant-id>",
    "ClientId": "<client-id>",
    "ClientSecret": "<client-secret>", // ONLY for local development (User Secrets)
    "CallbackPath": "/signin-oidc",
    "SignedOutCallbackPath": "/signout-callback-oidc"
  },
  "ManagedIdentity": {
    "Enabled": true, // Set to true in Azure, false for local development
    "ClientId": "<user-assigned-mi-client-id>", // REQUIRED: User-Assigned MI Client ID
    "UseManagedIdentityForGraph": true // Use Managed Identity for Graph API calls
  },
  "Authorization": {
    "AdminGroupId": "<Entra-Admin-Group-Object-Id>",
    "AdminGroupName": "Entra-Admin",
    "SupportGroupId": "<Entra-Support-Group-Object-Id>",
    "SupportGroupName": "Entra-Support",
    "RequireGroupMembership": true,
    "CacheGroupMembershipMinutes": 5
  },
  "GraphApi": {
    "BaseUrl": "https://graph.microsoft.com/v1.0",
    "Scopes": ["https://graph.microsoft.com/.default"],
    "BatchSize": 999,
    "RetryAttempts": 3,
    "RetryDelaySeconds": 2
  },
  "KeyVault": {
    "VaultUri": "https://<your-keyvault-name>.vault.azure.net/",
    "UseManagedIdentity": true
  },
  "Cache": {
    "ExpirationMinutes": 60,
    "EnableAutoRefresh": false,
    "AutoRefreshIntervalMinutes": 30
  },
  "Features": {
    "EnableDelete": true,
    "EnableExport": true,
    "EnableBulkOperations": false,
    "AllowSupportRefresh": false
  },
  "Session": {
    "IdleTimeoutMinutes": 30,
    "AbsoluteTimeoutHours": 8
  },
  "Monitoring": {
    "ApplicationInsights": {
      "ConnectionString": "" // Leave empty to use Managed Identity
    }
  }
}
```

### 6.3 Configuration Implementation - Works Locally and in Azure

**DESIGN PRINCIPLE**: Same configuration code works in both local development and Azure production. DefaultAzureCredential automatically detects the environment!

```csharp
// Program.cs - Unified configuration loading for local AND Azure
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

// ============================================================================
// CONFIGURATION LOADING ORDER
// ============================================================================
// By default, .NET loads in this order:
// 1. appsettings.json (base configuration)
// 2. appsettings.{Environment}.json (e.g., appsettings.Development.json)
// 3. User Secrets (Development only) - WE ADD THIS EXPLICITLY
// 4. Environment variables
// 5. Command-line arguments

// ============================================================================
// STEP 1: Add User Secrets (Development - HIGHEST PRIORITY for secrets)
// ============================================================================
if (builder.Environment.IsDevelopment())
{
    // Load secrets.json (managed via: dotnet user-secrets set "Key" "Value")
    builder.Configuration.AddUserSecrets<Program>(optional: true);
    
    // IMPORTANT: Map User Secrets to Environment Variables for DefaultAzureCredential
    // DefaultAzureCredential looks for AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, AZURE_TENANT_ID
    // in environment variables, so we map them from configuration
    var clientId = builder.Configuration["AZURE_CLIENT_ID"];
    var clientSecret = builder.Configuration["AZURE_CLIENT_SECRET"];
    var tenantId = builder.Configuration["AZURE_TENANT_ID"];
    
    if (!string.IsNullOrEmpty(clientId))
    {
        Environment.SetEnvironmentVariable("AZURE_CLIENT_ID", clientId);
        Environment.SetEnvironmentVariable("AZURE_CLIENT_SECRET", clientSecret);
        Environment.SetEnvironmentVariable("AZURE_TENANT_ID", tenantId);
    }
    
    // OPTIONAL: Also check for local.settings.json (common in Azure Functions)
    builder.Configuration.AddJsonFile("local.settings.json", optional: true, reloadOnChange: true);
}

// ============================================================================
// STEP 2: Environment variables (ALWAYS)
// ============================================================================
builder.Configuration.AddEnvironmentVariables();

// ============================================================================
// STEP 3: Create DefaultAzureCredential (Works locally AND in Azure!)
// ============================================================================
// Local Development: Uses Azure CLI (`az login`) or Visual Studio credentials
// Azure Production: Uses User-Assigned Managed Identity
var managedIdentityClientId = builder.Configuration["ManagedIdentity:ClientId"];

var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    // Specify User-Assigned MI Client ID (ignored locally, used in Azure)
    ManagedIdentityClientId = managedIdentityClientId,
    
    // Local development: Try these in order
    ExcludeInteractiveBrowserCredential = true,  // Don't prompt user
    
    // Production: Will use ManagedIdentityCredential
    // Development: Will use AzureCliCredential or VisualStudioCredential
});

// Register credential for dependency injection
builder.Services.AddSingleton<TokenCredential>(credential);

// ============================================================================
// STEP 4: Azure Key Vault (Production - loads secrets via Managed Identity)
// ============================================================================
// Local Development: If KeyVault:VaultUri is set, uses Azure CLI credentials
// Azure Production: Uses User-Assigned Managed Identity
var keyVaultUri = builder.Configuration["KeyVault:VaultUri"];
if (!string.IsNullOrEmpty(keyVaultUri))
{
    try
    {
        builder.Configuration.AddAzureKeyVault(
            new Uri(keyVaultUri),
            credential);  // Same credential works locally and in Azure!
        
        builder.Logging.AddConsole().SetMinimumLevel(LogLevel.Information);
        var logger = builder.Services.BuildServiceProvider().GetRequiredService<ILogger<Program>>();
        logger.LogInformation("✅ Key Vault configuration loaded from: {VaultUri}", keyVaultUri);
    }
    catch (Exception ex)
    {
        // Non-fatal: Key Vault might not be accessible locally (that's OK!)
        builder.Logging.AddConsole();
        var logger = builder.Services.BuildServiceProvider().GetRequiredService<ILogger<Program>>();
        logger.LogWarning(ex, "⚠️  Could not load Key Vault. Using local configuration.");
    }
}

// ============================================================================
// STEP 5: Configure Options Pattern
// ============================================================================
builder.Services.Configure<AzureAdOptions>(
    builder.Configuration.GetSection("AzureAd"));
builder.Services.Configure<GraphApiOptions>(
    builder.Configuration.GetSection("GraphApi"));
builder.Services.Configure<ManagedIdentityOptions>(
    builder.Configuration.GetSection("ManagedIdentity"));

// ============================================================================
// LOG CONFIGURATION SOURCE (Helpful for debugging)
// ============================================================================
if (builder.Environment.IsDevelopment())
{
    var config = builder.Configuration;
    Console.WriteLine("=== Configuration Sources (Priority Order) ===");
    Console.WriteLine($"1. User Secrets: {(config.GetSection("AzureAd:ClientSecret").Exists() ? "✅ Loaded" : "❌ Not found")}");
    Console.WriteLine($"2. local.settings.json: {(File.Exists("local.settings.json") ? "✅ Found" : "❌ Not found")}");
    Console.WriteLine($"3. appsettings.Development.json: ✅ Loaded");
    Console.WriteLine($"4. Environment Variables: ✅ Available");
    Console.WriteLine($"5. Key Vault: {(!string.IsNullOrEmpty(keyVaultUri) ? $"✅ {keyVaultUri}" : "❌ Not configured")}");
    Console.WriteLine("===========================================");
}
```

### 6.3.1 Local Development Setup Guide

**Step 1: Install Prerequisites**

```bash
# Install .NET 10 SDK
# Download from: https://dotnet.microsoft.com/download/dotnet/10.0

# Install Azure CLI (for DefaultAzureCredential local auth)
# Windows: Download from https://aka.ms/installazurecliwindows
# macOS: brew install azure-cli
# Linux: curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login to Azure (this is what DefaultAzureCredential will use locally!)
az login

# Set your subscription (if you have multiple)
az account set --subscription "<your-subscription-id>"

# Verify
az account show
```

**Step 2: Initialize User Secrets (secrets.json)**

```bash
# Navigate to your project directory
cd entra-id-app-portal/entra-id-app-portal.Web

# Initialize user secrets
dotnet user-secrets init

# Add your secrets (these stay LOCAL and never get committed to git!)
dotnet user-secrets set "AzureAd:ClientSecret" "your-dev-client-secret-here"
dotnet user-secrets set "Authorization:AdminGroupId" "your-admin-group-object-id"
dotnet user-secrets set "Authorization:SupportGroupId" "your-support-group-object-id"

# Optional: Set Managed Identity to disabled for local dev
dotnet user-secrets set "ManagedIdentity:Enabled" "false"
dotnet user-secrets set "ManagedIdentity:UseManagedIdentityForGraph" "false"

# List all secrets (to verify)
dotnet user-secrets list
```

**Where are User Secrets stored?**
- Windows: `%APPDATA%\Microsoft\UserSecrets\<user_secrets_id>\secrets.json`
- macOS/Linux: `~/.microsoft/usersecrets/<user_secrets_id>/secrets.json`

**Step 3: Create appsettings.Development.json (optional local overrides)**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "<your-dev-tenant-id>",
    "ClientId": "<your-dev-app-registration-client-id>",
    // ClientSecret is in secrets.json - NOT here!
    "CallbackPath": "/signin-oidc",
    "SignedOutCallbackPath": "/signout-callback-oidc"
  },
  "ManagedIdentity": {
    "Enabled": false,  // Use Azure CLI credentials locally
    "UseManagedIdentityForGraph": false  // Use delegated auth locally
  },
  "GraphApi": {
    "BaseUrl": "https://graph.microsoft.com/v1.0",
    "Scopes": ["https://graph.microsoft.com/.default"],
    "BatchSize": 999,
    "RetryAttempts": 3,
    "RetryDelaySeconds": 2
  },
  "KeyVault": {
    "VaultUri": "",  // Leave empty for local dev (use secrets.json instead)
    "UseManagedIdentity": false
  }
}
```

**Step 4: Create local.settings.json (alternative to appsettings.Development.json)**

```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureAd__TenantId": "<your-dev-tenant-id>",
    "AzureAd__ClientId": "<your-dev-app-registration-client-id>",
    "ManagedIdentity__Enabled": "false"
  }
}
```

**Step 5: Add to .gitignore (CRITICAL!)**

```gitignore
# User-specific files
*.user
*.userosscache
*.suo

# User Secrets
secrets.json
**/secrets.json

# Local settings
local.settings.json
appsettings.local.json

# Environment files
.env
.env.local
```

### 6.3.2 Local Debugging Experience

**F5 Debugging - It Just Works!**

```csharp
// When you press F5 in Visual Studio:
// 1. Application starts in Development environment
// 2. Loads appsettings.json (base config)
// 3. Loads appsettings.Development.json (dev overrides)
// 4. Loads secrets.json via User Secrets (sensitive values)
// 5. Loads local.settings.json (if present)
// 6. Applies environment variables (if any)
// 7. DefaultAzureCredential uses Azure CLI credentials (from `az login`)
// 8. All your configuration is ready!

// NO CHANGES NEEDED FOR PRODUCTION - Same code works in Azure!
```

**Debugging Configuration Issues:**

```csharp
// Add to Program.cs (Development only) for troubleshooting
if (builder.Environment.IsDevelopment())
{
    var config = builder.Configuration;
    
    // Check what values are loaded
    Console.WriteLine($"Tenant ID: {config["AzureAd:TenantId"]}");
    Console.WriteLine($"Client ID: {config["AzureAd:ClientId"]}");
    Console.WriteLine($"Has Client Secret: {!string.IsNullOrEmpty(config["AzureAd:ClientSecret"])}");
    Console.WriteLine($"MI Enabled: {config["ManagedIdentity:Enabled"]}");
    Console.WriteLine($"MI Client ID: {config["ManagedIdentity:ClientId"] ?? "Not set"}");
    
    // Test DefaultAzureCredential
    try
    {
        var testToken = await credential.GetTokenAsync(
            new TokenRequestContext(new[] { "https://graph.microsoft.com/.default" }));
        Console.WriteLine($"✅ DefaultAzureCredential working! Token expires: {testToken.ExpiresOn}");
    }
    catch (Exception ex)
    {
        Console.WriteLine($"❌ DefaultAzureCredential failed: {ex.Message}");
        Console.WriteLine("💡 Tip: Run 'az login' to authenticate locally");
    }
}
```

### 6.3.3 Configuration Comparison: Local vs Azure

| Aspect | Local Development | Azure Production |
|--------|-------------------|------------------|
| **Auth Method** | Azure CLI (`az login`) | User-Assigned Managed Identity |
| **Secrets Storage** | secrets.json (User Secrets) | Azure Key Vault |
| **Configuration** | appsettings.Development.json | App Service Configuration |
| **DefaultAzureCredential** | Uses AzureCliCredential | Uses ManagedIdentityCredential |
| **Key Vault Access** | Optional (via Azure CLI) | Required (via Managed Identity) |
| **Client Secret** | In secrets.json (dev only) | Not used (MI instead) |
| **Configuration Changes** | **NONE - Same code!** | **NONE - Same code!** |

### 6.3.4 Environment Detection Logic

```csharp
// The application automatically detects where it's running!
public class EnvironmentService
{
    private readonly IWebHostEnvironment _environment;
    private readonly IConfiguration _configuration;
    
    public bool IsRunningInAzure()
    {
        // Azure App Service sets this environment variable
        return !string.IsNullOrEmpty(Environment.GetEnvironmentVariable("WEBSITE_INSTANCE_ID"));
    }
    
    public bool IsUsingManagedIdentity()
    {
        return _configuration.GetValue<bool>("ManagedIdentity:Enabled") && IsRunningInAzure();
    }
    
    public bool IsLocalDevelopment()
    {
        return _environment.IsDevelopment() && !IsRunningInAzure();
    }
    
    public string GetConfigurationSource()
    {
        if (IsLocalDevelopment())
            return "secrets.json + appsettings.Development.json + Azure CLI";
        else if (IsRunningInAzure())
            return "Azure Key Vault + App Service Configuration + Managed Identity";
        else
            return "appsettings.json + Environment Variables";
    }
}
```

---

## 6.4 Managed Identity & Azure Default Credential - Best Practices

### Overview

**CRITICAL SECURITY REQUIREMENT**: This application MUST use **User-Assigned Managed Identity** to access Azure resources and Microsoft Graph API. NO client secrets or certificates should be used in production.

**Benefits of User-Assigned Managed Identity**:

- ✅ No secrets to manage or rotate
- ✅ Automatic credential management by Azure
- ✅ **Independent lifecycle** - not deleted when App Service is deleted
- ✅ **Reusable** - same identity across Web App, API, Functions, etc.
- ✅ **Pre-created** - set up permissions before deploying application
- ✅ Enhanced security posture
- ✅ Simplified multi-resource deployments (Aspire AppHost + Services)
- ✅ Audit trail of all resource access

**Why User-Assigned vs System-Assigned Managed Identity?**

| Aspect | System-Assigned MI | User-Assigned MI (Recommended) |
|--------|-------------------|--------------------------------|
| **Lifecycle** | Tied to resource (deleted with App Service) | Independent (survives resource deletion) |
| **Reusability** | One per resource | Shared across multiple resources |
| **Setup Timing** | Created after resource exists | Created before resource deployment |
| **Permission Management** | Per resource | Centralized (one set of permissions) |
| **Aspire Compatibility** | Must configure each service separately | Assign same MI to all services |
| **Best For** | Single resource scenarios | Multi-resource apps, Aspire, microservices |
| **Configuration Complexity** | Simple (no Client ID needed) | Requires Client ID in config |

**Recommendation for this project**: Use **User-Assigned MI** because:
1. .NET Aspire may deploy multiple services (Web, API, Background workers)
2. All services need same permissions (Graph API, Key Vault)
3. Easier to manage permissions centrally
4. Can pre-configure everything before first deployment
5. Safer for blue-green deployments (MI not deleted with old slot)

### 6.4.1 DefaultAzureCredential Chain

Use `DefaultAzureCredential` from Azure.Identity SDK - it provides a credential chain that tries authentication methods in order:

**Authentication Chain Order**:

1. **EnvironmentCredential** - Reads account info from environment variables
2. **WorkloadIdentityCredential** - Azure Kubernetes Service workload identity
3. **ManagedIdentityCredential** - User-Assigned Managed Identity (Azure resources)
4. **SharedTokenCacheCredential** - Uses cached credentials from developer tools
5. **VisualStudioCredential** - Uses signed-in Visual Studio account
6. **VisualStudioCodeCredential** - Uses signed-in VS Code Azure account
7. **AzureCliCredential** - Uses Azure CLI logged-in account
8. **AzurePowerShellCredential** - Uses Azure PowerShell logged-in account
9. **AzureDeveloperCliCredential** - Uses Azure Developer CLI (azd)
10. **InteractiveBrowserCredential** - Opens browser for interactive login (disabled by default)

**Result**: Developers use their own credentials locally (Azure CLI, VS Code), production uses Managed Identity automatically!

### 6.4.2 Required NuGet Packages

```xml
<PackageReference Include="Azure.Identity" Version="1.13.*" />
<PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.6.*" />
<PackageReference Include="Azure.Extensions.AspNetCore.Configuration.Secrets" Version="1.3.*" />
<PackageReference Include="Microsoft.Graph" Version="5.*" />
<PackageReference Include="Microsoft.Identity.Web" Version="3.*" />
<PackageReference Include="Microsoft.Identity.Web.MicrosoftGraph" Version="3.*" />
<PackageReference Include="Microsoft.Identity.Web.TokenCache" Version="3.*" />
```

### 6.4.3 Configuration Setup with DefaultAzureCredential

```csharp
// Program.cs
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Microsoft.Graph;

var builder = WebApplication.CreateBuilder(args);

// Create DefaultAzureCredential instance
// This will work both locally (using developer credentials) and in Azure (using User-Assigned MI)
var managedIdentityClientId = builder.Configuration["ManagedIdentity:ClientId"];

var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    // CRITICAL: Specify User-Assigned Managed Identity Client ID
    // This is REQUIRED for User-Assigned MI (without it, will try System-Assigned)
    ManagedIdentityClientId = managedIdentityClientId,
    
    // Exclude credentials you don't want to use (optional)
    ExcludeInteractiveBrowserCredential = true, // Don't open browser
    ExcludeSharedTokenCacheCredential = true,   // Optional: exclude if not needed

    // Enable logging for troubleshooting
    Diagnostics =
    {
        LoggedHeaderNames = { "x-ms-request-id" },
        LoggedQueryParameters = { "api-version" },
        IsLoggingContentEnabled = true
    }
});

// Register credential as singleton
builder.Services.AddSingleton(credential);

// 1. Load configuration from Azure Key Vault using Managed Identity
var keyVaultUri = builder.Configuration["KeyVault:VaultUri"];
if (!string.IsNullOrEmpty(keyVaultUri))
{
    builder.Configuration.AddAzureKeyVault(
        new Uri(keyVaultUri),
        credential); // Uses DefaultAzureCredential
}

// 2. Configure Application Insights with Managed Identity
builder.Services.AddApplicationInsightsTelemetry(options =>
{
    // Connection string from Key Vault or config
    options.ConnectionString = builder.Configuration["ApplicationInsights:ConnectionString"];
});

// 3. Configure Microsoft Graph with Managed Identity
builder.Services.AddMicrosoftGraph(options =>
{
    options.Scopes = new[] { "https://graph.microsoft.com/.default" };
})
.ConfigurePrimaryHttpMessageHandler(() => new SocketsHttpHandler
{
    PooledConnectionLifetime = TimeSpan.FromMinutes(2)
});

// 4. Register Graph Service Client with Managed Identity
builder.Services.AddSingleton<GraphServiceClient>(sp =>
{
    var useManagedIdentity = builder.Configuration.GetValue<bool>("ManagedIdentity:UseManagedIdentityForGraph");

    if (useManagedIdentity)
    {
        // Production: Use Managed Identity
        var graphCredential = credential;
        return new GraphServiceClient(graphCredential,
            new[] { "https://graph.microsoft.com/.default" });
    }
    else
    {
        // Development: Use delegated credentials (on behalf of user)
        // This is configured separately via Microsoft.Identity.Web
        var graphClient = sp.GetRequiredService<GraphServiceClient>();
        return graphClient;
    }
});

// 5. Configure Redis Cache with Managed Identity (if using Azure Cache for Redis)
var redisConnectionString = builder.Configuration["Redis:ConnectionString"];
if (!string.IsNullOrEmpty(redisConnectionString))
{
    builder.Services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = redisConnectionString;
        // Azure Cache for Redis with Managed Identity requires Azure.Extensions.AspNetCore.Configuration.Secrets
    });
}

// 6. Register Key Vault client for runtime secret access
builder.Services.AddSingleton<SecretClient>(sp =>
{
    var vaultUri = builder.Configuration["KeyVault:VaultUri"];
    return new SecretClient(new Uri(vaultUri), credential);
});
```

### 6.4.4 Graph API Service Implementation with Managed Identity

```csharp
// Services/GraphService.cs
using Azure.Identity;
using Microsoft.Graph;
using Microsoft.Graph.Models;

public interface IGraphService
{
    Task<IEnumerable<Application>> GetApplicationsAsync(int top = 999);
    Task<IEnumerable<string>> GetUserGroupsAsync(string userId);
    Task DeleteApplicationAsync(string applicationId);
}

public class GraphService : IGraphService
{
    private readonly GraphServiceClient _graphClient;
    private readonly ILogger<GraphService> _logger;
    private readonly IConfiguration _configuration;

    public GraphService(
        TokenCredential credential, // DefaultAzureCredential injected
        IConfiguration configuration,
        ILogger<GraphService> logger)
    {
        _configuration = configuration;
        _logger = logger;

        // Create Graph client with Managed Identity
        _graphClient = new GraphServiceClient(
            credential,
            new[] { "https://graph.microsoft.com/.default" });
    }

    public async Task<IEnumerable<Application>> GetApplicationsAsync(int top = 999)
    {
        try
        {
            var applications = new List<Application>();

            // Get applications with batching
            var response = await _graphClient.Applications
                .GetAsync(requestConfiguration =>
                {
                    requestConfiguration.QueryParameters.Top = top;
                    requestConfiguration.QueryParameters.Select = new[]
                    {
                        "id", "appId", "displayName", "createdDateTime",
                        "passwordCredentials", "keyCredentials"
                    };
                    requestConfiguration.QueryParameters.Orderby = new[] { "displayName" };
                });

            if (response?.Value != null)
            {
                applications.AddRange(response.Value);

                // Handle pagination
                var pageIterator = PageIterator<Application, ApplicationCollectionResponse>
                    .CreatePageIterator(
                        _graphClient,
                        response,
                        app =>
                        {
                            applications.Add(app);
                            return true; // Continue iterating
                        });

                await pageIterator.IterateAsync();
            }

            _logger.LogInformation("Retrieved {Count} applications from Graph API", applications.Count);
            return applications;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to retrieve applications from Graph API");
            throw;
        }
    }

    public async Task<IEnumerable<string>> GetUserGroupsAsync(string userId)
    {
        try
        {
            var groups = new List<string>();

            var response = await _graphClient.Users[userId]
                .MemberOf
                .GraphGroup
                .GetAsync();

            if (response?.Value != null)
            {
                groups.AddRange(response.Value.Select(g => g.Id ?? string.Empty));
            }

            return groups;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to retrieve user groups for {UserId}", userId);
            throw;
        }
    }

    public async Task DeleteApplicationAsync(string applicationId)
    {
        try
        {
            await _graphClient.Applications[applicationId].DeleteAsync();
            _logger.LogInformation("Deleted application {ApplicationId}", applicationId);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to delete application {ApplicationId}", applicationId);
            throw;
        }
    }
}
```

### 6.4.5 Key Vault Access Pattern

```csharp
// Services/SecretService.cs
using Azure.Security.KeyVault.Secrets;
using Azure.Identity;

public interface ISecretService
{
    Task<string> GetSecretAsync(string secretName);
}

public class SecretService : ISecretService
{
    private readonly SecretClient _secretClient;
    private readonly ILogger<SecretService> _logger;

    public SecretService(
        TokenCredential credential, // DefaultAzureCredential injected
        IConfiguration configuration,
        ILogger<SecretService> logger)
    {
        _logger = logger;
        var vaultUri = configuration["KeyVault:VaultUri"];

        _secretClient = new SecretClient(
            new Uri(vaultUri!),
            credential); // Uses Managed Identity in Azure, developer creds locally
    }

    public async Task<string> GetSecretAsync(string secretName)
    {
        try
        {
            KeyVaultSecret secret = await _secretClient.GetSecretAsync(secretName);
            _logger.LogInformation("Retrieved secret {SecretName} from Key Vault", secretName);
            return secret.Value;
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to retrieve secret {SecretName} from Key Vault", secretName);
            throw;
        }
    }
}
```

### 6.4.6 Local Development - Quick Start

**IMPORTANT**: Developers should use a **Development User-Assigned MI** (not personal Azure credentials) because:
- ❌ Personal accounts lack Graph API application permissions (e.g., `Application.Read.All`)
- ❌ These are admin-only permissions that can't be granted to individual developers
- ✅ Development MI has the same permissions as production MI
- ✅ All developers share the same identity and permissions
- ✅ No individual admin permissions needed

**TL;DR - 3 Steps to Start Debugging Locally:**

```bash
# Step 1: Get Development MI Client ID (ask your IT admin or get it yourself)
az login
DEV_MI_CLIENT_ID=$(az identity show \
  --name entra-portal-dev-identity \
  --resource-group dev-rg \
  --query clientId -o tsv)

# Step 2: Initialize and set user secrets
cd entra-id-app-portal.Web
dotnet user-secrets init
dotnet user-secrets set "ManagedIdentity:ClientId" "$DEV_MI_CLIENT_ID"
dotnet user-secrets set "ManagedIdentity:Enabled" "true"
dotnet user-secrets set "ManagedIdentity:UseManagedIdentityForGraph" "true"
dotnet user-secrets set "Authorization:AdminGroupId" "your-admin-group-id"
dotnet user-secrets set "Authorization:SupportGroupId" "your-support-group-id"

# Step 3: Press F5 in Visual Studio - It just works!
```

**That's it!** The application will:
- ✅ Load configuration from secrets.json (User Secrets)
- ✅ Use **Development Managed Identity** for Graph API (same permissions as production!)
- ✅ Use the same code as production (no conditional logic needed)
- ✅ Work without individual developer admin permissions

### 6.4.6.1 Development Managed Identity Setup (IT Admin Task)

**One-Time Setup by IT Admin:**

```bash
# ============================================================================
# STEP 1: Create Development User-Assigned Managed Identity
# ============================================================================
az identity create \
  --name entra-portal-dev-identity \
  --resource-group dev-rg \
  --location eastus

# Get the IDs
DEV_MI_CLIENT_ID=$(az identity show --name entra-portal-dev-identity --resource-group dev-rg --query clientId -o tsv)
DEV_MI_PRINCIPAL_ID=$(az identity show --name entra-portal-dev-identity --resource-group dev-rg --query principalId -o tsv)

echo "Development MI Client ID: $DEV_MI_CLIENT_ID"
echo "Development MI Principal ID: $DEV_MI_PRINCIPAL_ID"

# ============================================================================
# STEP 2: Grant Graph API Permissions (Same as Production MI)
# ============================================================================
```

```powershell
# PowerShell: Grant Microsoft Graph application permissions
Connect-MgGraph -Scopes "Application.ReadWrite.All", "AppRoleAssignment.ReadWrite.All"

$devMIPrincipalId = "<dev-mi-principal-id>"
$graphSp = Get-MgServicePrincipal -Filter "displayName eq 'Microsoft Graph'"

# Grant same permissions as production
$permissions = @(
    "Application.Read.All",        # Or Application.ReadWrite.All
    "Directory.Read.All",
    "GroupMember.Read.All"
)

foreach ($permission in $permissions) {
    $appRole = $graphSp.AppRoles | Where-Object { $_.Value -eq $permission }
    
    if ($appRole) {
        New-MgServicePrincipalAppRoleAssignment `
            -ServicePrincipalId $devMIPrincipalId `
            -PrincipalId $devMIPrincipalId `
            -ResourceId $graphSp.Id `
            -AppRoleId $appRole.Id
        
        Write-Host "✅ Granted $permission to Development MI" -ForegroundColor Green
    }
}

Write-Host "`n✅ Development Managed Identity configured!" -ForegroundColor Green
Write-Host "Client ID for developers: $((Get-MgServicePrincipal -ServicePrincipalId $devMIPrincipalId).AppId)"
```

```bash
# ============================================================================
# STEP 3: Grant Development MI Access to Development Key Vault (Optional)
# ============================================================================
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $DEV_MI_PRINCIPAL_ID \
  --scope /subscriptions/<sub-id>/resourceGroups/dev-rg/providers/Microsoft.KeyVault/vaults/dev-keyvault

# ============================================================================
# STEP 4: Grant Developers Permission to Use the Development MI
# ============================================================================
# Developers need "Managed Identity Operator" role to use the MI
az role assignment create \
  --role "Managed Identity Operator" \
  --assignee "developer1@company.com" \
  --scope /subscriptions/<sub-id>/resourcegroups/dev-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/entra-portal-dev-identity

# Repeat for each developer OR use Azure AD group:
az role assignment create \
  --role "Managed Identity Operator" \
  --assignee-object-id "<developers-aad-group-object-id>" \
  --assignee-principal-type Group \
  --scope /subscriptions/<sub-id>/resourcegroups/dev-rg/providers/Microsoft.ManagedIdentity/userAssignedIdentities/entra-portal-dev-identity

echo "✅ Development MI setup complete!"
echo "Share this Client ID with developers: $DEV_MI_CLIENT_ID"
```

### 6.4.6.2 Developer Machine Setup

**Per Developer (One-Time Setup):**

```bash
# ============================================================================
# Prerequisites
# ============================================================================
# 1. .NET 10 SDK installed
# 2. Azure CLI installed
# 3. Visual Studio 2022+ or VS Code

# ============================================================================
# STEP 1: Clone Repository
# ============================================================================
git clone <repository-url>
cd entra-id-app-portal

# ============================================================================
# STEP 2: Login to Azure
# ============================================================================
# This is needed to authenticate to Azure and use the Development MI
az login

# Verify you're logged in and can access the subscription
az account show

# ============================================================================
# STEP 3: Get Development MI Client ID
# ============================================================================
# Option A: Ask your IT admin for the Client ID
# Option B: Get it yourself (if you have access)
DEV_MI_CLIENT_ID=$(az identity show \
  --name entra-portal-dev-identity \
  --resource-group dev-rg \
  --query clientId -o tsv)

echo "Development MI Client ID: $DEV_MI_CLIENT_ID"

# ============================================================================
# STEP 4: Configure User Secrets
# ============================================================================
cd entra-id-app-portal.Web

# Initialize user secrets
dotnet user-secrets init

# Set Development MI configuration
dotnet user-secrets set "ManagedIdentity:ClientId" "$DEV_MI_CLIENT_ID"
dotnet user-secrets set "ManagedIdentity:Enabled" "true"
dotnet user-secrets set "ManagedIdentity:UseManagedIdentityForGraph" "true"

# Set group IDs (ask your admin or get from Azure Portal)
dotnet user-secrets set "Authorization:AdminGroupId" "<admin-group-object-id>"
dotnet user-secrets set "Authorization:SupportGroupId" "<support-group-object-id>"

# Optional: Set for user authentication (if you want to test login)
dotnet user-secrets set "AzureAd:ClientSecret" "<dev-app-registration-secret>"

# List secrets to verify
dotnet user-secrets list

# ============================================================================
# STEP 5: Press F5 to Debug!
# ============================================================================
# Open solution in Visual Studio or VS Code and press F5
# The application will:
# 1. Load configuration from secrets.json
# 2. Use DefaultAzureCredential with Development MI Client ID
# 3. Authenticate using the Development MI (via your Azure CLI login)
# 4. Have the same permissions as production!
```

**How It Works - Development MI with Environment Variables:**

Since Managed Identities only work on Azure resources (not local machines), we use **EnvironmentCredential** to simulate MI locally:

```bash
# IT Admin: Get the Development MI credentials
# The Development MI needs a way to authenticate locally
# We'll use environment variables that DefaultAzureCredential understands

# Option A: Use Service Principal with same permissions as Development MI
# This is what we recommend because it works seamlessly

# IT Admin creates a Service Principal for local development
az ad sp create-for-rbac \
  --name "entra-portal-dev-sp" \
  --skip-assignment

# Output includes:
# - appId (Client ID)
# - password (Client Secret)
# - tenant

# Grant this SP the SAME Graph API permissions as Development MI (using PowerShell)
# See section 6.4.6.1 above - same PowerShell script, different principal ID
```

**Simplified Developer Setup:**

```bash
# After IT Admin creates the Development Service Principal
# Developers set these in User Secrets:

cd entra-id-app-portal.Web
dotnet user-secrets init

# Set the Service Principal credentials (acts as Development MI)
dotnet user-secrets set "AZURE_CLIENT_ID" "<dev-sp-client-id>"
dotnet user-secrets set "AZURE_CLIENT_SECRET" "<dev-sp-client-secret>"
dotnet user-secrets set "AZURE_TENANT_ID" "<tenant-id>"

# Set other required configuration
dotnet user-secrets set "Authorization:AdminGroupId" "<admin-group-id>"
dotnet user-secrets set "Authorization:SupportGroupId" "<support-group-id>"

# Press F5 - DefaultAzureCredential automatically uses EnvironmentCredential!
```

**How DefaultAzureCredential Works with This Approach:**

```
Developer presses F5
  ↓
DefaultAzureCredential created
  ↓
1. Checks EnvironmentCredential first
   Looks for: AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, AZURE_TENANT_ID
   ✅ Found in User Secrets! (mapped to environment variables)
  ↓
2. Uses Service Principal authentication
   Authenticates as the Development SP (which has MI-like permissions)
  ↓
3. Acquires tokens from Azure AD
   Token has Application permissions (Application.Read.All, etc.)
  ↓
4. Calls Graph API successfully!
   Same permissions as Production MI!
```

**Authentication Options for Local Development:**

| Method | Setup | Permissions | Recommended |
|--------|-------|-------------|-------------|
| **Development MI** | Use environment vars | Same as production | ✅ **Best** - Production parity |
| **Service Principal** | Client ID + Secret | Same as production | ✅ **Best** - Easy to use |
| **Azure CLI** | `az login` | Your personal permissions | ⚠️ Requires admin permissions |
| **Visual Studio** | Tools → Options | Your personal permissions | ⚠️ Requires admin permissions |

**Local Configuration Priority (Highest to Lowest):**
1. Environment variables (`AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID`) - **DefaultAzureCredential checks these first!**
2. secrets.json (User Secrets) - **Store credentials here** (they get mapped to env vars)
3. local.settings.json (if present)
4. appsettings.Development.json
5. appsettings.json

### 6.4.6.3 Summary: Development MI Approach

**Setup Overview:**

| Step | Who | What | Why |
|------|-----|------|-----|
| 1. Create Development Service Principal | IT Admin | `az ad sp create-for-rbac` | Acts as Development MI locally |
| 2. Grant Graph API Permissions | IT Admin | PowerShell script | Same permissions as Production MI |
| 3. Share credentials with developers | IT Admin | Securely share Client ID + Secret | Via secure channel (Key Vault, password manager) |
| 4. Set User Secrets | Each Developer | `dotnet user-secrets set` | Store credentials locally (not in git) |
| 5. Press F5 | Each Developer | Debug | DefaultAzureCredential uses Service Principal |

**Benefits:**
- ✅ Developers have same permissions as production (no surprises!)
- ✅ No personal admin permissions needed for developers
- ✅ Works seamlessly with DefaultAzureCredential (no code changes)
- ✅ Easy to rotate credentials (just update Service Principal secret)
- ✅ Easy to revoke access (delete Service Principal)
- ✅ Same code works in Azure with actual Managed Identity

**Alternative (If you want to use actual MI locally):**
Run development in Azure (VM, Container Instance, or App Service slot) with Development MI assigned. This gives you true MI authentication but requires running in Azure.

### 6.4.7 Azure Managed Identity Setup

#### Step 1: Create User-Assigned Managed Identity

**IMPORTANT**: Create the User-Assigned Managed Identity BEFORE deploying the application. This allows you to configure permissions in advance.

**Azure CLI:**

```bash
# Create User-Assigned Managed Identity
az identity create \
    --name entra-portal-identity \
    --resource-group <resource-group-name> \
    --location <location>

# Capture the output - you'll need these values:
# - clientId (Application/Client ID) - for configuration
# - principalId (Object/Principal ID) - for granting permissions
# - id (Resource ID) - for assigning to App Service

# Get the IDs (if you need to retrieve them later)
az identity show \
    --name entra-portal-identity \
    --resource-group <resource-group-name> \
    --query "{clientId: clientId, principalId: principalId, id: id}" \
    --output json
```

**Output Example:**
```json
{
  "clientId": "12345678-1234-1234-1234-123456789abc",
  "principalId": "87654321-4321-4321-4321-cba987654321",
  "id": "/subscriptions/<sub-id>/resourcegroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/entra-portal-identity"
}
```

**Azure Portal:**

1. Navigate to Azure Portal
2. Search for "Managed Identities"
3. Click "+ Create"
4. Select subscription, resource group, region
5. Name: `entra-portal-identity`
6. Click "Review + Create"
7. After creation, copy:
   - **Client ID** (for application configuration)
   - **Principal ID** (for permission grants)
   - **Resource ID** (for App Service assignment)

#### Step 1b: Assign User-Assigned Managed Identity to App Service

**Azure CLI:**

```bash
# Assign the User-Assigned MI to App Service
az webapp identity assign \
    --name <app-name> \
    --resource-group <resource-group-name> \
    --identities /subscriptions/<sub-id>/resourcegroups/<rg>/providers/Microsoft.ManagedIdentity/userAssignedIdentities/entra-portal-identity

# Verify assignment
az webapp identity show \
    --name <app-name> \
    --resource-group <resource-group-name>
```

**Azure Portal:**

1. Navigate to App Service → Identity
2. Click "User assigned" tab
3. Click "+ Add"
4. Select your managed identity: `entra-portal-identity`
5. Click "Add"

**IMPORTANT**: Add the Client ID to your configuration:
```json
{
  "ManagedIdentity": {
    "Enabled": true,
    "ClientId": "12345678-1234-1234-1234-123456789abc"
  }
}
```

#### Step 2: Grant User-Assigned Managed Identity Access to Key Vault

**Using Azure RBAC (Recommended):**

```bash
# Get the Principal ID of the User-Assigned MI (if you don't have it)
PRINCIPAL_ID=$(az identity show \
    --name entra-portal-identity \
    --resource-group <resource-group-name> \
    --query principalId \
    --output tsv)

echo "Principal ID: $PRINCIPAL_ID"

# Assign Key Vault Secrets User role to the User-Assigned MI
az role assignment create \
    --role "Key Vault Secrets User" \
    --assignee $PRINCIPAL_ID \
    --scope /subscriptions/<subscription-id>/resourceGroups/<rg-name>/providers/Microsoft.KeyVault/vaults/<kv-name>

# Verify role assignment
az role assignment list \
    --assignee $PRINCIPAL_ID \
    --scope /subscriptions/<subscription-id>/resourceGroups/<rg-name>/providers/Microsoft.KeyVault/vaults/<kv-name>
```

**Using Access Policies (Legacy - not recommended):**

```bash
# Grant Key Vault access using access policies
az keyvault set-policy \
    --name <key-vault-name> \
    --object-id $PRINCIPAL_ID \
    --secret-permissions get list
```

#### Step 3: Grant User-Assigned Managed Identity Access to Microsoft Graph

**CRITICAL**: The User-Assigned Managed Identity needs Graph API application permissions to call Microsoft Graph on behalf of the application (not user context).

```powershell
# Connect to Microsoft Graph with admin privileges
Connect-MgGraph -Scopes "Application.ReadWrite.All", "AppRoleAssignment.ReadWrite.All"

# Get the User-Assigned Managed Identity Service Principal (by Principal/Object ID)
# Get this from: az identity show --name entra-portal-identity --query principalId -o tsv
$managedIdentityPrincipalId = "<user-assigned-mi-principal-id>"  # NOT the Client ID!

# Get the Service Principal object
$managedIdentitySp = Get-MgServicePrincipal -ServicePrincipalId $managedIdentityPrincipalId

Write-Host "Found User-Assigned MI: $($managedIdentitySp.DisplayName)"
Write-Host "Principal ID: $($managedIdentitySp.Id)"

# Get Microsoft Graph Service Principal
$graphSp = Get-MgServicePrincipal -Filter "displayName eq 'Microsoft Graph'"

Write-Host "Found Microsoft Graph SP: $($graphSp.Id)"

# Define required Graph API permissions (Application permissions, not delegated)
$permissions = @(
    "Application.Read.All",        # Or Application.ReadWrite.All for delete operations
    "Directory.Read.All",          # Read directory data
    "GroupMember.Read.All"         # Read group memberships
)

# Assign permissions to the User-Assigned Managed Identity
foreach ($permission in $permissions) {
    $appRole = $graphSp.AppRoles | Where-Object { $_.Value -eq $permission }

    if ($appRole) {
        try {
            New-MgServicePrincipalAppRoleAssignment `
                -ServicePrincipalId $managedIdentityPrincipalId `
                -PrincipalId $managedIdentityPrincipalId `
                -ResourceId $graphSp.Id `
                -AppRoleId $appRole.Id

            Write-Host "✅ Granted $permission to User-Assigned Managed Identity" -ForegroundColor Green
        }
        catch {
            Write-Host "⚠️  Failed to grant $permission : $_" -ForegroundColor Yellow
        }
    }
    else {
        Write-Host "❌ Permission $permission not found" -ForegroundColor Red
    }
}

Write-Host "`n✅ User-Assigned Managed Identity configured successfully!" -ForegroundColor Green
Write-Host "Client ID to use in configuration: $($managedIdentitySp.AppId)"
```

**Alternative using Azure CLI and REST API:**

```bash
# Get the User-Assigned MI Principal ID
PRINCIPAL_ID=$(az identity show \
    --name entra-portal-identity \
    --resource-group <resource-group-name> \
    --query principalId \
    --output tsv)

echo "User-Assigned MI Principal ID: $PRINCIPAL_ID"

# Get Microsoft Graph Service Principal ID
GRAPH_SP_ID=$(az ad sp list \
    --display-name "Microsoft Graph" \
    --query "[0].id" \
    --output tsv)

echo "Microsoft Graph SP ID: $GRAPH_SP_ID"

# Function to grant permission
grant_graph_permission() {
    local PERMISSION=$1
    
    # Get the app role ID for the permission
    APP_ROLE_ID=$(az ad sp show \
        --id $GRAPH_SP_ID \
        --query "appRoles[?value=='$PERMISSION'].id" \
        --output tsv)
    
    if [ -z "$APP_ROLE_ID" ]; then
        echo "❌ Permission $PERMISSION not found"
        return 1
    fi
    
    # Assign the role using Graph API REST
    az rest --method POST \
        --uri "https://graph.microsoft.com/v1.0/servicePrincipals/$PRINCIPAL_ID/appRoleAssignments" \
        --headers "Content-Type=application/json" \
        --body "{
            \"principalId\": \"$PRINCIPAL_ID\",
            \"resourceId\": \"$GRAPH_SP_ID\",
            \"appRoleId\": \"$APP_ROLE_ID\"
        }"
    
    echo "✅ Granted $PERMISSION to User-Assigned Managed Identity"
}

# Grant required permissions
grant_graph_permission "Application.Read.All"
grant_graph_permission "Directory.Read.All"
grant_graph_permission "GroupMember.Read.All"

echo "✅ All permissions granted successfully!"
```

#### Step 4: Grant User-Assigned Managed Identity Access to Other Azure Resources

**Get the Principal ID (if not already set):**

```bash
PRINCIPAL_ID=$(az identity show \
    --name entra-portal-identity \
    --resource-group <resource-group-name> \
    --query principalId \
    --output tsv)
```

**Application Insights:**

```bash
# Assign Monitoring Metrics Publisher role to User-Assigned MI
az role assignment create \
    --role "Monitoring Metrics Publisher" \
    --assignee $PRINCIPAL_ID \
    --scope /subscriptions/<subscription-id>/resourceGroups/<rg-name>/providers/Microsoft.Insights/components/<app-insights-name>

# Verify
az role assignment list --assignee $PRINCIPAL_ID --scope /subscriptions/<subscription-id>/resourceGroups/<rg-name>/providers/Microsoft.Insights/components/<app-insights-name>
```

**Azure Cache for Redis (if using):**

```bash
# Assign Redis Cache Contributor role to User-Assigned MI
az role assignment create \
    --role "Redis Cache Contributor" \
    --assignee $PRINCIPAL_ID \
    --scope /subscriptions/<subscription-id>/resourceGroups/<rg-name>/providers/Microsoft.Cache/Redis/<redis-name>
```

**Summary of User-Assigned MI Setup:**

```bash
# Quick reference script
MI_NAME="entra-portal-identity"
RG_NAME="<your-resource-group>"

# Get IDs
CLIENT_ID=$(az identity show --name $MI_NAME --resource-group $RG_NAME --query clientId -o tsv)
PRINCIPAL_ID=$(az identity show --name $MI_NAME --resource-group $RG_NAME --query principalId -o tsv)
MI_RESOURCE_ID=$(az identity show --name $MI_NAME --resource-group $RG_NAME --query id -o tsv)

echo "User-Assigned Managed Identity Details:"
echo "  Name: $MI_NAME"
echo "  Client ID (for config): $CLIENT_ID"
echo "  Principal ID (for permissions): $PRINCIPAL_ID"
echo "  Resource ID (for assignment): $MI_RESOURCE_ID"
```

### 6.4.8 Troubleshooting DefaultAzureCredential

**Enable Detailed Logging:**

```csharp
// Program.cs
using Azure.Core.Diagnostics;

// Enable Azure SDK logging
using AzureEventSourceListener listener = AzureEventSourceListener.CreateConsoleLogger(EventLevel.Verbose);

// Or log to file
using AzureEventSourceListener listener = AzureEventSourceListener.CreateTraceLogger(EventLevel.Verbose);
```

**Common Issues:**

1. **Local Development: "DefaultAzureCredential failed to retrieve a token"**

   - Solution: Run `az login` or sign in to Visual Studio/VS Code
   - Verify: `az account show`

2. **Azure: "ManagedIdentityCredential authentication failed"**

   - Verify User-Assigned MI is created and assigned to App Service
   - Check App Service → Identity → User assigned tab
   - Verify `ManagedIdentity:ClientId` is set correctly in configuration
   - Verify permissions: Check Key Vault RBAC assignments for the Principal ID
   - Check Graph API permissions: Ensure app roles are assigned to the Principal ID
   - Verify the MI Resource ID is correctly assigned to the App Service

3. **"AADSTS700016: Application not found in the directory"**

   - Managed Identity principal not granted Graph API permissions
   - Follow Step 3 above to grant permissions

4. **Key Vault Access Denied**
   - Verify: `az keyvault secret show --name <secret-name> --vault-name <vault-name>`
   - Grant access: Follow Step 2 above

**Diagnostic Code:**

```csharp
// Services/DiagnosticService.cs
public async Task TestDefaultAzureCredential()
{
    try
    {
        var credential = new DefaultAzureCredential();

        // Try to acquire token for Graph API
        var tokenContext = new TokenRequestContext(
            new[] { "https://graph.microsoft.com/.default" });

        var token = await credential.GetTokenAsync(tokenContext);

        _logger.LogInformation("Successfully acquired token. Expires: {ExpiresOn}",
            token.ExpiresOn);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Failed to acquire token with DefaultAzureCredential");
    }
}
```

### 6.4.9 Best Practices Summary

✅ **DO**:

- Use `DefaultAzureCredential` for all Azure resource access
- **Create User-Assigned Managed Identity BEFORE deploying** (allows pre-configuration)
- **Specify `ManagedIdentityClientId` in DefaultAzureCredential options**
- Use Azure CLI login (`az login`) for local development
- Store MI Client ID in configuration (it's not sensitive)
- Log credential acquisition for troubleshooting
- Use RBAC over access policies when possible
- Document MI Principal ID for permission grants
- Test with User-Assigned MI in staging environment first
- Reuse same User-Assigned MI across related resources (Web App, Functions, etc.)

❌ **DON'T**:

- Don't use System-Assigned MI (use User-Assigned for better lifecycle management)
- Don't store client secrets in code or appsettings.json (production)
- Don't use client secrets in production (use Managed Identity)
- Don't hardcode tenant IDs in code (use configuration)
- Don't expose Managed Identity Principal/Object IDs in client-side code
- Don't grant excessive permissions (principle of least privilege)
- Don't skip testing Managed Identity before production deployment
- Don't forget to assign the User-Assigned MI to the App Service
- Don't confuse Client ID (for config) with Principal ID (for permissions)

### 6.4.10 Environment-Specific Configuration

**IMPORTANT**: Same configuration structure, different values and sources!

**Development - secrets.json (User Secrets):**

```json
{
  "AzureAd": {
    "ClientSecret": "your-dev-app-registration-secret"
  },
  "ManagedIdentity": {
    "Enabled": false,  // Use Azure CLI credentials instead
    "UseManagedIdentityForGraph": false  // Use delegated auth
  },
  "Authorization": {
    "AdminGroupId": "your-admin-group-object-id",
    "SupportGroupId": "your-support-group-object-id"
  }
}
```

**Development - appsettings.Development.json (Non-sensitive values):**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Debug",
      "Microsoft.AspNetCore": "Information"
    }
  },
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "your-dev-tenant-id",
    "ClientId": "your-dev-app-client-id",
    "CallbackPath": "/signin-oidc"
  },
  "KeyVault": {
    "VaultUri": "",  // Empty = Don't use Key Vault locally
    "UseManagedIdentity": false
  },
  "GraphApi": {
    "BaseUrl": "https://graph.microsoft.com/v1.0"
  }
}
```

**Production - Azure Key Vault (Secrets stored securely):**

```bash
# Store in Key Vault (accessed via Managed Identity - NO secrets in config!)
az keyvault secret set --vault-name <vault-name> --name "Authorization--AdminGroupId" --value "<admin-group-id>"
az keyvault secret set --vault-name <vault-name> --name "Authorization--SupportGroupId" --value "<support-group-id>"
```

**Production - Azure App Service Configuration (App Settings):**

```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "production-tenant-id",
    "ClientId": "production-app-client-id",
    "CallbackPath": "/signin-oidc"
  },
  "ManagedIdentity": {
    "Enabled": true,  // Use User-Assigned Managed Identity
    "ClientId": "user-assigned-mi-client-id",
    "UseManagedIdentityForGraph": true
  },
  "KeyVault": {
    "VaultUri": "https://prod-keyvault.vault.azure.net/",
    "UseManagedIdentity": true
  }
}
```

**Key Differences:**

| Setting | Development | Production |
|---------|-------------|------------|
| **Secrets Source** | secrets.json (local file) | Azure Key Vault |
| **Authentication** | Azure CLI (`az login`) | User-Assigned Managed Identity |
| **ClientSecret** | In secrets.json | NOT used (MI instead) |
| **ManagedIdentity:Enabled** | `false` | `true` |
| **KeyVault:VaultUri** | Empty or commented out | Production Key Vault URL |
| **Code Changes** | **ZERO!** | **ZERO!** |

**The Magic**: DefaultAzureCredential automatically selects the right credential:
- 🏠 **Local**: Uses Azure CLI credential from `az login`
- ☁️ **Azure**: Uses User-Assigned Managed Identity

**Same code, different credentials - zero configuration changes needed!**

---

## 7. Authentication & Authorization

### 7.1 Authentication Requirements

**CRITICAL**: The portal MUST require authentication. No features or pages should be accessible to unauthenticated users.

**Authentication Flow**:

- Use Microsoft Identity Platform (Microsoft.Identity.Web)
- Implement **OAuth 2.0 Authorization Code Flow with PKCE** (Proof Key for Code Exchange)
- Redirect unauthenticated users to Microsoft login page
- Acquire access tokens for Microsoft Graph API
- Support automatic token refresh
- Implement proper logout functionality

**User Experience**:

- Landing page redirects immediately to login if not authenticated
- Display user name and email in header after authentication
- Provide logout button in navigation bar
- Show "Access Denied" page for authenticated users without proper group membership

### 7.2 Group-Based Authorization

**Entra ID Security Groups**: Authorization is based on Entra ID group membership, not individual role assignments.

| Entra ID Group    | Role          | Access Level              | Permissions                                                                                                                                                |
| ----------------- | ------------- | ------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Entra-Admin**   | Administrator | Full Access (Super Users) | - View all app registrations<br>- Apply all filters<br>- Export data<br>- Refresh cache<br>- **Delete app registrations**<br>- Access all features         |
| **Entra-Support** | Reader        | Read-Only Access          | - View all app registrations<br>- Apply all filters<br>- Export data<br>- **NO** delete permissions<br>- **NO** refresh permissions (optional restriction) |

**Access Control Rules**:

1. User MUST be authenticated to access any portal feature
2. User MUST be a member of at least one authorized group (Entra-Admin OR Entra-Support)
3. Users not in either group see "Access Denied" message
4. Group membership is checked on every request (cached for performance)
5. Delete operations require "Entra-Admin" group membership

### 7.3 Required App Registration Configuration

**App Registration Setup** (Azure Portal):

1. Create new App Registration in Entra ID
2. Configure **Authentication**:

   - Platform: Web
   - Redirect URI: `https://<your-app-url>/signin-oidc`
   - Logout URL: `https://<your-app-url>/signout-callback-oidc`
   - Enable ID tokens and access tokens
   - Supported account types: Single tenant

3. Configure **API Permissions** (Delegated):

   - `User.Read` - Sign in and read user profile
   - `Application.Read.All` - Read all applications (or Application.ReadWrite.All for delete)
   - `Directory.Read.All` - Read directory data
   - `GroupMember.Read.All` - Read group memberships for the signed-in user

4. **Optional - Application Permissions** (if using app-only flow for background jobs):

   - `Application.Read.All` or `Application.ReadWrite.All`
   - `Directory.Read.All`

5. **Token Configuration**:

   - Add optional claim: `groups` (emit security groups in token)
   - OR configure group claims to return security groups
   - Alternative: Use `GroupMember.Read.All` to query groups via API

6. **Certificates & Secrets**:
   - Create client secret (store in Key Vault)
   - OR use certificate-based authentication (recommended for production)

### 7.4 Authorization Implementation

#### Option A: Groups in Token (Recommended for < 200 groups)

Configure app registration to include groups in token claims:

```csharp
// Program.cs
builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd"))
    .EnableTokenAcquisitionToCallDownstreamApi()
    .AddMicrosoftGraph(builder.Configuration.GetSection("GraphApi"))
    .AddInMemoryTokenCaches();

builder.Services.AddAuthorization(options =>
{
    // Require authentication for all pages
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();

    // Policy for Admin operations (Entra-Admin group)
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireClaim("groups", "<Entra-Admin-Group-ObjectId>"));

    // Policy for any authorized user (Admin OR Support)
    options.AddPolicy("AuthorizedUser", policy =>
        policy.RequireAssertion(context =>
            context.User.HasClaim(c => c.Type == "groups" &&
                (c.Value == "<Entra-Admin-Group-ObjectId>" ||
                 c.Value == "<Entra-Support-Group-ObjectId>"))));
});

// Apply authorization globally
builder.Services.AddRazorPages()
    .AddMicrosoftIdentityUI();

builder.Services.AddControllers(options =>
{
    // Require authenticated user for all API endpoints
    var policy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
    options.Filters.Add(new AuthorizeFilter(policy));
});
```

#### Option B: Query Groups via Graph API (Recommended for many groups)

```csharp
// Services/IUserAuthorizationService.cs
public interface IUserAuthorizationService
{
    Task<bool> IsUserInGroupAsync(string groupName);
    Task<bool> IsAdministratorAsync();
    Task<bool> IsAuthorizedUserAsync();
    Task<UserRole> GetUserRoleAsync();
}

// Services/UserAuthorizationService.cs
public class UserAuthorizationService : IUserAuthorizationService
{
    private readonly GraphServiceClient _graphClient;
    private readonly IHttpContextAccessor _httpContextAccessor;
    private readonly IConfiguration _configuration;
    private readonly IMemoryCache _cache;

    public async Task<bool> IsAdministratorAsync()
    {
        var adminGroupId = _configuration["Authorization:AdminGroupId"];
        return await IsUserInGroupAsync(adminGroupId);
    }

    public async Task<bool> IsAuthorizedUserAsync()
    {
        var adminGroupId = _configuration["Authorization:AdminGroupId"];
        var supportGroupId = _configuration["Authorization:SupportGroupId"];

        return await IsUserInGroupAsync(adminGroupId) ||
               await IsUserInGroupAsync(supportGroupId);
    }

    private async Task<bool> IsUserInGroupAsync(string groupId)
    {
        var userId = _httpContextAccessor.HttpContext?.User
            .FindFirst("http://schemas.microsoft.com/identity/claims/objectidentifier")?.Value;

        if (string.IsNullOrEmpty(userId)) return false;

        // Check cache first (5 minute TTL)
        var cacheKey = $"UserGroup_{userId}_{groupId}";
        if (_cache.TryGetValue(cacheKey, out bool isMember))
            return isMember;

        try
        {
            // Check group membership via Graph API
            var result = await _graphClient.Users[userId]
                .CheckMemberGroups
                .PostAsync(new CheckMemberGroupsPostRequestBody
                {
                    GroupIds = new List<string> { groupId }
                });

            isMember = result?.Value?.Contains(groupId) ?? false;

            // Cache result
            _cache.Set(cacheKey, isMember, TimeSpan.FromMinutes(5));

            return isMember;
        }
        catch
        {
            return false;
        }
    }

    public async Task<UserRole> GetUserRoleAsync()
    {
        if (await IsAdministratorAsync())
            return UserRole.Administrator;

        if (await IsAuthorizedUserAsync())
            return UserRole.Reader;

        return UserRole.Unauthorized;
    }
}

public enum UserRole
{
    Unauthorized,
    Reader,
    Administrator
}
```

#### Controller Authorization

```csharp
// Controllers/AppRegistrationsController.cs
[Authorize] // Require authentication
[ApiController]
[Route("api/[controller]")]
public class AppRegistrationsController : ControllerBase
{
    private readonly IUserAuthorizationService _authService;

    [HttpGet]
    public async Task<IActionResult> GetAll()
    {
        // Any authenticated user in authorized groups
        if (!await _authService.IsAuthorizedUserAsync())
            return Forbid();

        // Return data
    }

    [HttpPost("refresh")]
    public async Task<IActionResult> RefreshCache()
    {
        // Only admins can refresh (optional - or allow support too)
        if (!await _authService.IsAdministratorAsync())
            return Forbid();

        // Refresh logic
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> Delete(string id)
    {
        // Only admins can delete
        if (!await _authService.IsAdministratorAsync())
            return Forbid();

        // Delete logic
    }
}
```

#### Blazor Component Authorization

```razor
@* Components/Pages/AppRegistrations.razor *@
@page "/appregistrations"
@attribute [Authorize] @* Require authentication *@
@inject IUserAuthorizationService AuthService

@if (!isAuthorized)
{
    <div class="alert alert-danger">
        <h4>Access Denied</h4>
        <p>You do not have permission to access this portal.</p>
        <p>Please contact your administrator to be added to the Entra-Admin or Entra-Support group.</p>
    </div>
    return;
}

@* Show delete button only for admins *@
@if (isAdmin)
{
    <button @onclick="DeleteApp" class="btn btn-danger">Delete</button>
}

@code {
    private bool isAuthorized = false;
    private bool isAdmin = false;

    protected override async Task OnInitializedAsync()
    {
        isAuthorized = await AuthService.IsAuthorizedUserAsync();
        isAdmin = await AuthService.IsAdministratorAsync();
    }
}
```

### 7.5 Configuration for Authorization

Update configuration to include group IDs:

```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "<tenant-id>",
    "ClientId": "<client-id>",
    "ClientSecret": "<client-secret>",
    "CallbackPath": "/signin-oidc",
    "SignedOutCallbackPath": "/signout-callback-oidc"
  },
  "Authorization": {
    "AdminGroupId": "<Entra-Admin-Group-Object-Id>",
    "AdminGroupName": "Entra-Admin",
    "SupportGroupId": "<Entra-Support-Group-Object-Id>",
    "SupportGroupName": "Entra-Support",
    "RequireGroupMembership": true
  },
  "GraphApi": {
    "BaseUrl": "https://graph.microsoft.com/v1.0",
    "Scopes": ["User.Read", "Application.Read.All", "GroupMember.Read.All"]
  }
}
```

### 7.6 Login/Logout Flow

**Login Flow**:

1. User navigates to application URL
2. Application detects unauthenticated user
3. Redirect to Microsoft login page (OAuth2 authorization endpoint)
4. User enters credentials and consents to permissions
5. Microsoft redirects back with authorization code
6. Application exchanges code for access token and ID token
7. Application validates group membership
8. If authorized, user gains access; otherwise, show "Access Denied"

**Logout Flow**:

1. User clicks logout button
2. Clear local session and cookies
3. Redirect to Microsoft logout endpoint
4. Microsoft clears session and redirects back to application
5. User sees login page

**Implementation**:

```csharp
// Program.cs
app.UseAuthentication();
app.UseAuthorization();

app.MapGet("/", () => Results.Redirect("/appregistrations"))
    .RequireAuthorization();

app.MapGet("/access-denied", () =>
    Results.Content(
        "<h1>Access Denied</h1><p>You must be a member of Entra-Admin or Entra-Support group.</p>",
        "text/html"))
    .AllowAnonymous();
```

### 7.7 Security Considerations

1. **Token Security**:

   - Store tokens securely (encrypted cookies or distributed cache)
   - Never expose tokens in client-side code
   - Use HTTPS only (enforce in production)

2. **Group Membership Caching**:

   - Cache group checks to reduce Graph API calls
   - Use short TTL (5 minutes) for cache
   - Clear cache on logout

3. **Session Management**:

   - Implement sliding session expiration (20-30 minutes)
   - Absolute session timeout (8 hours)
   - Force re-authentication for sensitive operations (delete)

4. **Audit Logging**:

   - Log all authentication attempts (success/failure)
   - Log authorization failures (unauthorized access attempts)
   - Log group membership checks
   - Log all delete operations with user context

5. **Error Handling**:
   - Never expose group IDs or internal details in error messages
   - Provide user-friendly "Access Denied" messages
   - Log detailed errors server-side for troubleshooting

---

## 8. Caching Strategy

### 8.1 Cache Requirements

- **In-Memory Cache**: Use `IMemoryCache` for single-instance deployments
- **Distributed Cache**: Use Redis (`IDistributedCache`) for multi-instance (production)
- **Cache Key**: `"AppRegistrations:All"`
- **Cache Invalidation**: Manual refresh only (no TTL)

### 8.2 Cache Service Interface

```csharp
public interface ICacheService
{
    Task<List<AppRegistrationDto>> GetAllAppsAsync();
    Task<AppRegistrationDto> GetAppByIdAsync(string id);
    Task RefreshCacheAsync(IProgress<RefreshStatus> progress = null);
    Task<CacheStatus> GetCacheStatusAsync();
    Task ClearCacheAsync();
    DateTime? GetLastRefreshTime();
}
```

### 8.3 Batch Loading Implementation

```csharp
public async Task RefreshCacheAsync(IProgress<RefreshStatus> progress = null)
{
    var allApps = new List<AppRegistrationDto>();
    var totalCount = 0;
    var processedCount = 0;

    // Get first batch to determine total count
    var firstBatch = await _graphService.GetApplicationsAsync(top: 999);
    allApps.AddRange(firstBatch.Value);
    processedCount = firstBatch.Value.Count;

    progress?.Report(new RefreshStatus
    {
        ProcessedApps = processedCount,
        TotalApps = totalCount,
        IsComplete = false
    });

    // Continue with pagination
    var nextLink = firstBatch.ODataNextLink;
    while (!string.IsNullOrEmpty(nextLink))
    {
        var batch = await _graphService.GetApplicationsAsync(nextLink);
        allApps.AddRange(batch.Value);
        processedCount += batch.Value.Count;

        progress?.Report(new RefreshStatus
        {
            ProcessedApps = processedCount,
            TotalApps = totalCount,
            IsComplete = false
        });

        nextLink = batch.ODataNextLink;
    }

    // Enrich with secrets/certificates
    await EnrichWithSecretsAsync(allApps, progress);

    // Store in cache
    await _cache.SetAsync("AppRegistrations:All", allApps);
    await _cache.SetAsync("AppRegistrations:LastRefresh", DateTime.UtcNow);

    progress?.Report(new RefreshStatus
    {
        ProcessedApps = processedCount,
        TotalApps = processedCount,
        IsComplete = true
    });
}
```

---

## 9. UI/UX Requirements

### 9.1 Layout

- **Header**:
  - App title/logo (left)
  - User info section (right):
    - User display name
    - User email (tooltip or dropdown)
    - User role badge (Admin or Support)
    - Logout button
  - All header elements visible on all pages
- **Navigation**: Side menu or top nav with sections:
  - Dashboard
  - All Applications
  - Expiring Secrets
  - Settings (future)
  - Role-based menu items (show delete only for Admins)
- **Main Content**: Data grid/table with filters panel

- **Footer**:
  - Copyright
  - Application version
  - Last refresh time
  - Environment indicator (Dev/Staging/Prod)

### 9.1.1 Login/Logout Experience

**Login Page** (if user is not authenticated):

- Option 1: Auto-redirect to Microsoft login (recommended)
- Option 2: Landing page with "Sign in with Microsoft" button
- Clean, professional design
- Show application logo and name
- Brief description of portal purpose

**Logout Confirmation** (optional):

- Confirm logout action
- Clear message about session ending
- Redirect to login page or public landing

**Access Denied Page** (authenticated but not in authorized group):

- Clear "Access Denied" heading
- Friendly message explaining the requirement
- List required groups: "You must be a member of Entra-Admin or Entra-Support"
- Contact information for requesting access
- User details shown (so they know which account they're using)
- Logout button available

### 9.2 App Registrations Page

**Components**:

1. **Toolbar**:

   - Search bar (prominent position)
   - Refresh button (with spinner when active)
   - Export button (dropdown: CSV, Excel, JSON)
   - Filter toggle button
   - Last refresh timestamp

2. **Filter Panel** (collapsible):

   - All filter options listed in FR-2
   - Apply/Clear buttons
   - Active filter badges

3. **Data Grid**:

   - Sortable columns
   - Row actions: View Details, Delete (icon buttons)
   - Visual indicators: Color-coded expiry status
   - Pagination controls
   - Items per page selector

4. **Loading States**:

   - Skeleton loader for initial load
   - Progress bar for refresh
   - Disabled state for controls during operations

5. **Empty States**:
   - No data: "No applications found. Click Refresh to load."
   - No search results: "No applications match your filters."

### 9.3 Expiring Secrets Dashboard

**Layout**:

- Summary cards at top:
  - Expired (red, count)
  - Critical < 7 days (orange, count)
  - Warning 7-30 days (yellow, count)
  - Healthy > 30 days (green, count)
- Grouped lists below cards:
  - Each group shows app name, secret name, expiry date, days remaining
  - Click to view details

### 9.4 Color Scheme

- **Primary**: Microsoft Blue (#0078D4)
- **Success**: Green (#107C10)
- **Warning**: Yellow/Orange (#FFB900, #FF8C00)
- **Danger**: Red (#D13438)
- **Neutral**: Grays for backgrounds and text

### 9.5 Responsive Breakpoints

- **Desktop**: > 1200px (full features)
- **Tablet**: 768px - 1200px (collapsible filters)
- **Mobile**: < 768px (hamburger menu, simplified grid)

---

## 10. Error Handling & Logging

### 10.1 Error Categories

1. **Graph API Errors**:

   - Unauthorized (401) → Redirect to login
   - Forbidden (403) → Show permissions error
   - Throttling (429) → Retry with backoff
   - Server errors (5xx) → Show retry option

2. **Application Errors**:
   - Cache miss → Trigger refresh
   - Configuration missing → Show setup error
   - Network timeout → Show network error

### 10.2 Logging Strategy

Use structured logging with log levels:

```csharp
// Trace: Detailed flow information
_logger.LogTrace("Filtering apps with criteria: {FilterOptions}", filterOptions);

// Debug: Diagnostic information
_logger.LogDebug("Fetched {Count} apps from cache", apps.Count);

// Information: General flow milestones
_logger.LogInformation("Cache refresh started by user {UserId}", userId);

// Warning: Unusual but expected situations
_logger.LogWarning("Graph API throttling detected, retrying in {Seconds}s", delay);

// Error: Expected errors
_logger.LogError(ex, "Failed to delete app registration {AppId}", appId);

// Critical: Unexpected errors requiring immediate attention
_logger.LogCritical(ex, "Failed to acquire Graph API token");
```

**What to Log**:

- All API calls (endpoint, duration, status code)
- Cache operations (hit/miss, refresh)
- User actions (filter applied, delete requested)
- Authentication events
- Errors and exceptions (with context)

**What NOT to Log**:

- Secrets, tokens, passwords
- Full API responses (may contain sensitive data)
- User PII (beyond user ID)

### 10.3 Exception Handling

```csharp
public class GlobalExceptionHandler : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext httpContext,
        Exception exception,
        CancellationToken cancellationToken)
    {
        var (statusCode, message) = exception switch
        {
            GraphServiceException gex => HandleGraphException(gex),
            UnauthorizedAccessException => (401, "Unauthorized"),
            ArgumentException argEx => (400, argEx.Message),
            _ => (500, "An unexpected error occurred")
        };

        httpContext.Response.StatusCode = statusCode;
        await httpContext.Response.WriteAsJsonAsync(new
        {
            error = message,
            traceId = Activity.Current?.Id
        }, cancellationToken);

        return true;
    }
}
```

---

## 11. Testing Strategy

### 11.1 Unit Tests

**Coverage Areas**:

- Service layer logic (filtering, transformation)
- Cache operations
- DTO mapping
- Validation logic

**Framework**: xUnit + FluentAssertions + Moq

**Example**:

```csharp
[Fact]
public async Task FilterByExpiring_Should_Return_Apps_Expiring_Within_Days()
{
    // Arrange
    var apps = CreateTestApps();
    var filter = new FilterOptions { ExpiringInDays = 30 };

    // Act
    var result = _service.ApplyFilters(apps, filter);

    // Assert
    result.Should().OnlyContain(a =>
        a.DaysUntilExpiry <= 30 && a.DaysUntilExpiry >= 0);
}
```

### 11.2 Integration Tests

**Coverage Areas**:

- Graph API integration (using test tenant)
- Cache integration
- **Authentication flow (OAuth2 flow)**
- **Authorization checks (group membership)**
- API endpoints (end-to-end)

**Framework**: WebApplicationFactory + xUnit

**Authentication Testing Strategy**:

- Use test users from development tenant
- Create test accounts in both Entra-Admin and Entra-Support groups
- Test unauthorized user (not in any group)
- Mock Graph API responses for group membership
- Test token expiration and refresh

**Authorization Test Cases**:

```csharp
[Theory]
[InlineData("admin-user", true)]  // Admin can delete
[InlineData("support-user", false)]  // Support cannot delete
public async Task Delete_Authorization_Tests(string userType, bool shouldSucceed)
{
    // Arrange: Set up authenticated user with specific group
    // Act: Attempt delete operation
    // Assert: Verify authorization outcome
}
```

### 11.3 BDD Tests (Optional)

**Framework**: SpecFlow

**Example Scenario**:

```gherkin
Feature: Application Registration Filtering

  Scenario: Filter apps with expiring secrets
    Given the following app registrations exist:
      | Name   | SecretExpiry |
      | App1   | 2025-12-01   |
      | App2   | 2025-01-01   |
    When I filter by expiring in 30 days
    Then I should see 1 application
    And it should be "App2"
```

### 11.4 UI Testing

**Framework**: bUnit (Blazor unit testing) or Playwright

**Coverage Areas**:

- Component rendering
- User interactions (click, input)
- Loading states
- Error states

---

## 12. Deployment Strategy

### 12.1 Azure Resources

| Resource                 | Purpose                                | SKU/Tier                    | Managed Identity Required          |
| ------------------------ | -------------------------------------- | --------------------------- | ---------------------------------- |
| **User-Assigned MI**     | Identity for accessing all resources   | N/A                         | ✅ **Create FIRST**                |
| Azure App Service        | Host web app and API                   | B1 (Basic) or S1 (Standard) | ✅ Assign User-Assigned MI         |
| Azure Key Vault          | Store secrets (NOT client secrets)     | Standard                    | ✅ Grant MI "Key Vault Secrets User" |
| Azure Cache for Redis    | Distributed cache (optional)           | Basic C0                    | ✅ Grant MI "Redis Cache Contributor" |
| Application Insights     | Monitoring and telemetry               | Pay-as-you-go               | ✅ Grant MI "Monitoring Metrics Publisher" |
| Azure Container Registry | Container images (if using containers) | Basic                       | ✅ Grant MI "AcrPull" role         |

**Managed Identity Setup**: 
1. Create User-Assigned Managed Identity (`entra-portal-identity`) **FIRST**
2. Grant it permissions to all Azure resources (Key Vault, Graph API, etc.)
3. Assign it to App Service/Container Apps
4. Configure `ManagedIdentity:ClientId` in application settings
5. No client secrets or connection strings in configuration

### 12.2 Deployment Options

**Option 1: Azure App Service (Direct Deploy)**

- Create User-Assigned Managed Identity FIRST
- Grant permissions to MI (Key Vault, Graph API)
- Build and publish from CI/CD pipeline
- Deploy as .NET application
- **Assign User-Assigned Managed Identity to App Service**
- Configure `ManagedIdentity:ClientId` in App Settings
- No secrets in deployment configuration

**Option 2: Container-based (Preferred for Aspire)**

- Build Docker images via Aspire
- Push to Azure Container Registry (using MI for authentication)
- Deploy to Azure Container Apps or AKS
- **Container Apps automatically get Managed Identity**
- Configure environment variables (no secrets)

### 12.3 CI/CD Pipeline

**GitHub Actions Workflow**:

```yaml
name: Deploy to Azure

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: "10.0.x"

      - name: Restore dependencies
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore --configuration Release

      - name: Test
        run: dotnet test --no-build --configuration Release

      - name: Publish
        run: dotnet publish --no-build --configuration Release --output ./publish

      - name: Deploy to Azure
        uses: azure/webapps-deploy@v2
        with:
          app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
          publish-profile: ${{ secrets.AZURE_WEBAPP_PUBLISH_PROFILE }}
          package: ./publish
```

### 12.4 Environment Configuration

**Development**:

- Use User Secrets for sensitive values (client secrets - temporary only)
- Use Azure CLI login (`az login`) for DefaultAzureCredential
- Point to development tenant
- `ManagedIdentity:Enabled = false`
- `ManagedIdentity:UseManagedIdentityForGraph = false`

**Staging**:

- Use Azure Key Vault (accessed via Managed Identity)
- Enable System-Assigned Managed Identity on App Service
- Grant MI access to Key Vault, Graph API, and other resources
- Separate test Entra ID tenant (or test apps)
- `ManagedIdentity:Enabled = true`
- `ManagedIdentity:UseManagedIdentityForGraph = true`

**Production**:

- **CRITICAL**: Use Managed Identity for ALL Azure resource access
- Use Azure Key Vault (accessed via Managed Identity)
- **NO client secrets in configuration**
- **NO connection strings with credentials**
- System-Assigned Managed Identity enabled on App Service
- MI granted Graph API application permissions
- MI granted Key Vault access (RBAC: "Key Vault Secrets User")
- MI granted Application Insights access
- Production tenant
- `ManagedIdentity:Enabled = true`
- `ManagedIdentity:UseManagedIdentityForGraph = true`

### 12.5 Entra ID Security Groups Setup

**PREREQUISITE**: Before deploying the application, the following Entra ID security groups must be created and configured.

#### Step-by-Step Group Setup

1. **Create Entra-Admin Group**:

   - Navigate to Azure Portal → Entra ID → Groups
   - Click "New group"
   - Group type: Security
   - Group name: `Entra-Admin`
   - Group description: "Administrator access to Entra ID App Portal - full permissions including delete"
   - Membership type: Assigned (or Dynamic User if using rules)
   - Add initial members (IT administrators)
   - Copy the **Object ID** (needed for configuration)

2. **Create Entra-Support Group**:

   - Navigate to Azure Portal → Entra ID → Groups
   - Click "New group"
   - Group type: Security
   - Group name: `Entra-Support`
   - Group description: "Read-only access to Entra ID App Portal - view and export only"
   - Membership type: Assigned
   - Add initial members (support staff)
   - Copy the **Object ID** (needed for configuration)

3. **Get Group Object IDs**:

   ```bash
   # Using Azure CLI
   az ad group show --group "Entra-Admin" --query objectId -o tsv
   az ad group show --group "Entra-Support" --query objectId -o tsv

   # Using PowerShell
   Get-AzureADGroup -SearchString "Entra-Admin" | Select-Object ObjectId
   Get-AzureADGroup -SearchString "Entra-Support" | Select-Object ObjectId
   ```

4. **Update Application Configuration**:

   - Add group Object IDs to Azure Key Vault:
     - Secret name: `Authorization--AdminGroupId`
     - Secret name: `Authorization--SupportGroupId`
   - OR update appsettings.json (for development only):

   ```json
   "Authorization": {
     "AdminGroupId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
     "SupportGroupId": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy"
   }
   ```

5. **Configure App Registration for Group Claims**:

   - Option A: Include groups in token (recommended for < 200 groups per user)

     - Navigate to App Registration → Token Configuration
     - Click "Add groups claim"
     - Select "Security groups"
     - For ID tokens, select "Group ID"
     - For Access tokens, select "Group ID"

   - Option B: Query groups via API (recommended if users have many groups)
     - Ensure `GroupMember.Read.All` permission is granted
     - Application will query groups at runtime

#### Group Management Best Practices

- **Naming Convention**: Use consistent naming (e.g., `Entra-Admin`, `Entra-Support`)
- **Documentation**: Document group purpose and members
- **Regular Audits**: Review group membership quarterly
- **Just-in-Time Access**: Consider using PIM (Privileged Identity Management) for admin group
- **Testing**: Always test with both admin and support users before production deployment

#### Troubleshooting Group Authorization

**Issue**: User gets "Access Denied" despite being in the group

- Verify user is actually in the correct group (check Entra ID)
- Check token claims (decode JWT token to verify groups are included)
- Clear application cache and have user re-login
- Verify Group Object IDs match in configuration
- Check group membership cache TTL (default 5 minutes)

**Issue**: Groups not appearing in token

- Check token configuration in app registration
- If using token-based groups, ensure token size limit is not exceeded (200 group limit)
- Consider switching to API-based group checking if user has many groups

---

## 13. Security Best Practices

1. **Authentication**:

   - **Require authentication for ALL pages and API endpoints** (no anonymous access)
   - Always use HTTPS (enforce with HSTS headers)
   - Implement OAuth 2.0 Authorization Code Flow with PKCE
   - Implement proper token refresh (automatic silent renewal)
   - Use Managed Identity in Azure for app-to-app auth (no secrets in code)
   - Secure session cookies (HttpOnly, Secure, SameSite=Strict)
   - Implement proper logout (clear session + Microsoft logout)

2. **Authorization**:

   - **Enforce group-based authorization** (Entra-Admin, Entra-Support)
   - Check group membership on every sensitive operation
   - Cache group membership checks (5 min TTL) to reduce API calls
   - Implement least privilege principle
   - Log all authorization failures
   - Audit logs for all sensitive operations (delete, refresh)
   - Display user-friendly "Access Denied" messages (never expose internal details)

3. **Data Protection**:

   - Never log sensitive data (tokens, secrets, passwords, PII)
   - Encrypt data in transit (TLS 1.3 or 1.2+)
   - Use Azure Key Vault for all secrets and certificates
   - **Development Service Principal credentials** in User Secrets only (never in git)
   - Never expose group Object IDs in client-side code or error messages
   - Sanitize all user inputs
   - Implement proper CORS policy
   - Rotate Development SP credentials regularly (quarterly recommended)

4. **API Security**:

   - Implement rate limiting (per user and global)
   - CORS policy (whitelist specific origins only)
   - Content Security Policy (CSP) headers
   - Anti-forgery tokens for state-changing operations
   - Validate all inputs (prevent injection attacks)
   - Implement request size limits
   - Use API versioning for future changes

5. **Session Management**:

   - Implement sliding session expiration (30 min idle timeout)
   - Absolute session timeout (8 hours)
   - Force logout on browser close (if required)
   - Clear cache on logout
   - Regenerate session ID after login

6. **Dependency Management**:

   - Keep all NuGet packages up-to-date
   - Monitor for security vulnerabilities (GitHub Dependabot)
   - Use automated security scanning in CI/CD
   - Review dependencies regularly
   - Pin package versions in production

7. **Error Handling & Information Disclosure**:

   - Never expose stack traces to users
   - Use generic error messages for users
   - Log detailed errors server-side only
   - Don't reveal system information in error responses
   - Implement custom error pages (401, 403, 404, 500)

---

## 14. Development Guidelines

### 14.1 Local Development Workflow

**First-Time Setup (One-time):**

```bash
# 1. Clone repository
git clone <repository-url>
cd entra-id-app-portal

# 2. Install dependencies
dotnet restore

# 3. Login to Azure (for DefaultAzureCredential)
az login

# 4. Initialize user secrets
cd entra-id-app-portal.Web
dotnet user-secrets init

# 5. Set required secrets
dotnet user-secrets set "AzureAd:ClientSecret" "your-dev-secret"
dotnet user-secrets set "Authorization:AdminGroupId" "admin-group-id"
dotnet user-secrets set "Authorization:SupportGroupId" "support-group-id"

# 6. Return to solution root
cd ..

# 7. Open in IDE
# Visual Studio: Open .sln file
# VS Code: code .
```

**Daily Development Workflow:**

```bash
# 1. Pull latest changes
git pull

# 2. Restore packages (if needed)
dotnet restore

# 3. Ensure Azure CLI is still logged in
az account show  # If expired, run: az login

# 4. Press F5 to debug!
# - Configuration loads automatically (secrets.json → appsettings.Development.json)
# - DefaultAzureCredential uses your Azure CLI credentials
# - Same code works in Azure production!
```

**Debugging Tips:**

- ✅ Set breakpoints in Visual Studio/VS Code as normal
- ✅ Use Hot Reload for rapid UI changes (Blazor)
- ✅ Check Console output for configuration diagnostics
- ✅ Use Browser DevTools for front-end debugging
- ✅ Application Insights works locally too (if configured)

**Common Issues & Solutions:**

| Issue | Solution |
|-------|----------|
| "DefaultAzureCredential failed" | Run `az login` and verify with `az account show` |
| "Configuration key not found" | Check secrets.json: `dotnet user-secrets list` |
| "Unauthorized Graph API call" | Ensure your Azure account has permissions in dev tenant |
| "Key Vault access denied" | Set `KeyVault:VaultUri` to empty string in appsettings.Development.json |

### 14.2 Code Standards

- Follow Microsoft C# coding conventions
- Use StyleCop for code analysis
- XML documentation for public APIs
- Meaningful names (no abbreviations)
- **Never commit secrets** - Use User Secrets for development

### 14.3 Git Workflow

- Feature branches: `feature/description`
- Bug fixes: `bugfix/description`
- Pull requests required for main branch
- Commit messages: Use conventional commits
- **Always verify .gitignore** excludes secrets.json and local.settings.json

### 14.4 Code Review Checklist

- [ ] Code follows style guidelines
- [ ] Unit tests added/updated
- [ ] **No hardcoded secrets or credentials**
- [ ] **No secrets in appsettings.json or appsettings.Development.json**
- [ ] Error handling implemented
- [ ] Logging added for important operations
- [ ] XML comments added
- [ ] No console warnings or errors
- [ ] Configuration works both locally and in Azure (test DefaultAzureCredential)
- [ ] User Secrets used for local sensitive data
- [ ] .gitignore properly excludes secrets

### 14.5 Performance Considerations

- Use async/await properly
- Avoid N+1 queries
- Implement pagination for large datasets
- Cache frequently accessed data
- Use connection pooling
- **Reuse TokenCredential instances** (register as Singleton)
- **Cache Graph API results** appropriately

---

## 15. Monitoring & Observability

### 15.1 Application Insights Telemetry

**Custom Events**:

- Cache refresh triggered
- App registration deleted
- Filter applied
- Export requested

**Custom Metrics**:

- Cache hit ratio
- API call duration
- Apps with expiring secrets count

**Dependencies**:

- Track all Graph API calls
- Monitor response times
- Track failures

### 15.2 Health Checks

Implement health check endpoints:

```csharp
builder.Services.AddHealthChecks()
    .AddCheck<GraphApiHealthCheck>("graph-api")
    .AddCheck<CacheHealthCheck>("cache");

app.MapHealthChecks("/health");
```

### 15.3 Alerts

Configure alerts for:

- High error rate (> 5% of requests)
- Slow response times (> 5s)
- Failed Graph API calls
- High memory usage

---

## 16. Future Enhancements (Post-MVP)

### Phase 2:

1. Email notifications for expiring secrets
2. Scheduled reports
3. Bulk operations (bulk delete, bulk update)
4. App registration creation/editing
5. Service Principal management

### Phase 3:

1. Advanced analytics dashboard
2. Usage insights (sign-in stats)
3. Compliance reporting
4. Custom alerts and thresholds
5. Multi-tenant support

### Phase 4:

1. Workflow automation (auto-rotation of secrets)
2. Approval workflows for sensitive operations
3. Integration with Azure DevOps/ServiceNow
4. Mobile app

---

## 17. Success Criteria

### Definition of Done:

- [ ] All FR-1 to FR-7 implemented and tested
- [ ] Unit test coverage > 80%
- [ ] Integration tests pass
- [ ] Successfully deployed to Azure
- [ ] Authentication working with Entra ID
- [ ] Responsive design works on mobile, tablet, desktop
- [ ] Documentation complete (README, API docs, user guide)
- [ ] Performance benchmarks met
- [ ] Security review passed
- [ ] User acceptance testing completed

### Developer Onboarding Checklist (Local Setup):

**Prerequisites:**
- [ ] .NET 10 SDK installed
- [ ] Visual Studio 2022+ or VS Code installed
- [ ] Git installed
- [ ] Obtain Development Service Principal credentials from IT Admin:
  - Client ID (Application ID)
  - Client Secret
  - Tenant ID
  - Group Object IDs

**Setup (< 5 minutes):**
- [ ] Clone repository: `git clone <repo-url>`
- [ ] Navigate to project: `cd entra-id-app-portal`
- [ ] Restore dependencies: `dotnet restore`
- [ ] Navigate to Web project: `cd entra-id-app-portal.Web`
- [ ] Initialize user secrets: `dotnet user-secrets init`
- [ ] Set Development Service Principal credentials (acts as Development MI):
  ```bash
  # These enable DefaultAzureCredential to work locally with MI-like permissions
  dotnet user-secrets set "AZURE_CLIENT_ID" "<dev-sp-client-id>"
  dotnet user-secrets set "AZURE_CLIENT_SECRET" "<dev-sp-client-secret>"
  dotnet user-secrets set "AZURE_TENANT_ID" "<tenant-id>"
  
  # Set group IDs for authorization
  dotnet user-secrets set "Authorization:AdminGroupId" "<admin-group-object-id>"
  dotnet user-secrets set "Authorization:SupportGroupId" "<support-group-object-id>"
  
  # Optional: For user authentication testing
  dotnet user-secrets set "AzureAd:ClientSecret" "<user-auth-app-secret>"
  ```
- [ ] Verify secrets: `dotnet user-secrets list`
- [ ] Press **F5** to debug - IT WORKS! 🎉

**Verification:**
- [ ] Application starts successfully
- [ ] DefaultAzureCredential uses Service Principal (check logs)
- [ ] Can access Graph API (loads app registrations) with Application permissions
- [ ] Can login with Azure AD credentials (user authentication)
- [ ] Configuration loads from secrets.json
- [ ] No secrets in appsettings.Development.json or committed to git

**What's Happening Under the Hood:**
1. ✅ DefaultAzureCredential reads `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID` from User Secrets
2. ✅ Uses EnvironmentCredential to authenticate as Development Service Principal
3. ✅ Service Principal has same Graph API permissions as Production MI
4. ✅ Application works exactly like production (same permissions!)
5. ✅ NO personal admin permissions needed for developers

**Troubleshooting:**
| Issue | Solution |
|-------|----------|
| "DefaultAzureCredential failed" | Verify `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`, `AZURE_TENANT_ID` are set in secrets |
| "Unauthorized Graph API call" | Contact IT Admin - Development SP may need Graph API permissions |
| "Configuration key not found" | Run `dotnet user-secrets list` to verify all required secrets |
| "Can't login to portal" | Check `AzureAd:ClientSecret` is set for user authentication |

**Tips:**
- ✅ Check configuration: `dotnet user-secrets list`
- ✅ Same code works in Azure with Managed Identity (no changes needed!)
- ✅ All developers use same Service Principal (easy to manage)
- ✅ Credentials stored locally only (never in git)

---

### Launch Checklist:

**Entra ID Setup:**

- [ ] **Entra ID Security Groups created** (Entra-Admin, Entra-Support)
- [ ] **Group Object IDs obtained and documented**
- [ ] **Test users added to security groups**
- [ ] Production Entra ID app registration created (for user authentication)
- [ ] **App registration configured with required delegated permissions**:
  - [ ] User.Read
  - [ ] Application.Read.All (or Application.ReadWrite.All)
  - [ ] Directory.Read.All
  - [ ] GroupMember.Read.All
  - [ ] Admin consent granted for all delegated permissions
- [ ] **Token configuration set up** (group claims or API-based)

**Managed Identity Setup (CRITICAL):**

- [ ] **User-Assigned Managed Identity created** (`entra-portal-identity`)
- [ ] **MI Client ID documented** (for application configuration)
- [ ] **MI Principal ID documented** (for permission grants)
- [ ] **MI Resource ID documented** (for App Service assignment)
- [ ] **Managed Identity granted Graph API application permissions**:
  - [ ] Application.Read.All (or Application.ReadWrite.All for delete)
  - [ ] Directory.Read.All
  - [ ] GroupMember.Read.All
  - [ ] Permissions granted via PowerShell (New-MgServicePrincipalAppRoleAssignment)
- [ ] **Managed Identity granted Key Vault access** (RBAC: "Key Vault Secrets User")
- [ ] **Managed Identity granted Application Insights access** (if using MI for telemetry)
- [ ] **User-Assigned MI assigned to App Service** (via Azure Portal or CLI)
- [ ] **ManagedIdentity:ClientId configured in App Settings**
- [ ] **Managed Identity tested** (acquire token successfully)

**Azure Resources:**

- [ ] Azure App Service provisioned
- [ ] Azure Key Vault provisioned
- [ ] Application Insights provisioned
- [ ] Azure Cache for Redis provisioned (optional)
- [ ] **Key Vault configured with secrets** (group IDs, not client secrets)
- [ ] **Group Object IDs added to Key Vault**

**Configuration:**

- [ ] **ManagedIdentity:Enabled = true in production**
- [ ] **ManagedIdentity:UseManagedIdentityForGraph = true**
- [ ] **NO client secrets in production configuration**
- [ ] Key Vault URI configured
- [ ] DefaultAzureCredential implemented in code

**Deployment:**

- [ ] CI/CD pipeline configured
- [ ] Publish profile or deployment credentials configured
- [ ] Environment variables set (non-sensitive only)

**Testing:**

- [ ] **Managed Identity token acquisition tested**
- [ ] **Graph API calls with Managed Identity tested**
- [ ] **Key Vault access with Managed Identity tested**
- [ ] **Authentication flow tested** (login/logout)
- [ ] **Authorization tested** (admin and support user access)
- [ ] **Delete operation tested** (admin only)

**Monitoring & Operations:**

- [ ] Monitoring and alerts configured
- [ ] Application Insights connected
- [ ] Audit logging enabled
- [ ] User roles assigned (group memberships)
- [ ] Backup and disaster recovery plan
- [ ] Runbook for common operations

**Security Review:**

- [ ] **Security review completed** (all endpoints require auth)
- [ ] **Verified: No client secrets in code or configuration**
- [ ] **Verified: All Azure resources accessed via Managed Identity**
- [ ] **Verified: Least privilege permissions granted**
- [ ] Penetration testing completed (if required)

---

## 18. Quick Reference: Authentication & Authorization

### Access Control Matrix

| Feature / Action            | Anonymous User       | Authenticated (No Group) | Entra-Support (Reader) | Entra-Admin (Administrator) |
| --------------------------- | -------------------- | ------------------------ | ---------------------- | --------------------------- |
| **Access Portal**           | ❌ Redirect to login | ❌ Access Denied page    | ✅ Allowed             | ✅ Allowed                  |
| **View App List**           | ❌                   | ❌                       | ✅                     | ✅                          |
| **Apply Filters**           | ❌                   | ❌                       | ✅                     | ✅                          |
| **View Details**            | ❌                   | ❌                       | ✅                     | ✅                          |
| **Export Data**             | ❌                   | ❌                       | ✅                     | ✅                          |
| **Refresh Cache**           | ❌                   | ❌                       | ❌ (optional: ✅)      | ✅                          |
| **Delete App Registration** | ❌                   | ❌                       | ❌                     | ✅                          |
| **View Expiring Secrets**   | ❌                   | ❌                       | ✅                     | ✅                          |

### Required Entra ID Groups

| Group Name        | Object ID Location                            | Purpose                    | Members                  |
| ----------------- | --------------------------------------------- | -------------------------- | ------------------------ |
| **Entra-Admin**   | Configuration: `Authorization:AdminGroupId`   | Full administrative access | IT Admins, DevOps leads  |
| **Entra-Support** | Configuration: `Authorization:SupportGroupId` | Read-only support access   | Support staff, Help desk |

### Required API Permissions (Delegated - for User Authentication)

**App Registration Delegated Permissions** (for signing in users):

```
✅ User.Read                    - Sign in and read user profile
✅ Application.Read.All          - Read applications (on behalf of user)
   Application.ReadWrite.All     - Required for delete operations
✅ Directory.Read.All            - Read directory data
✅ GroupMember.Read.All          - Read user's group memberships
```

### Required API Permissions (Application - for Managed Identity)

**Managed Identity Application Permissions** (for accessing Graph API without user context):

```
✅ Application.Read.All          - Read all applications (minimum)
   Application.ReadWrite.All     - Required for delete operations
✅ Directory.Read.All            - Read directory data
✅ GroupMember.Read.All          - Read group memberships

⚠️ CRITICAL: These must be assigned to the Managed Identity Service Principal
   using PowerShell: New-MgServicePrincipalAppRoleAssignment
```

### Managed Identity vs User Authentication

| Aspect              | User Authentication (Delegated)    | Managed Identity (Application)    |
| ------------------- | ---------------------------------- | --------------------------------- |
| **Purpose**         | Sign in users to web portal        | Access Graph API on behalf of app |
| **Credential Type** | OAuth2 tokens (user context)       | Managed Identity (app context)    |
| **Permission Type** | Delegated permissions              | Application permissions           |
| **Setup**           | App Registration + user consent    | System MI + PowerShell grant      |
| **Use Case**        | User login, group membership check | Load app registrations from Graph |

### Authentication Flow Summary

```
1. User visits portal URL
   ↓
2. Check if authenticated?
   ├─ No → Redirect to Microsoft login
   │        ↓
   │        User logs in with Entra ID credentials
   │        ↓
   │        Acquire tokens (ID token + Access token)
   │        ↓
   └─ Yes → Continue
   ↓
3. Check group membership (Entra-Admin OR Entra-Support)?
   ├─ Yes → Grant access with appropriate permissions
   └─ No → Show "Access Denied" page
```

### Configuration Checklist

**User Authentication (App Registration):**

- [ ] `AzureAd:TenantId` - Your Entra ID tenant ID
- [ ] `AzureAd:ClientId` - App registration client ID (for user sign-in)
- [ ] `AzureAd:ClientSecret` - App registration secret (DEV ONLY - User Secrets)
- [ ] App Registration: Redirect URI configured
- [ ] App Registration: Delegated permissions granted + admin consent
- [ ] App Registration: Token configuration (group claims OR API access)

**Managed Identity (Production):**

- [ ] `ManagedIdentity:Enabled` - Set to true in production
- [ ] `ManagedIdentity:ClientId` - Set to User-Assigned MI Client ID
- [ ] `ManagedIdentity:UseManagedIdentityForGraph` - Set to true in production
- [ ] User-Assigned Managed Identity created
- [ ] User-Assigned MI assigned to App Service
- [ ] Managed Identity granted Graph API application permissions
- [ ] Managed Identity granted Key Vault access

**Authorization:**

- [ ] `Authorization:AdminGroupId` - Object ID of Entra-Admin group
- [ ] `Authorization:SupportGroupId` - Object ID of Entra-Support group
- [ ] Group IDs stored in Key Vault (accessed via Managed Identity)

**Azure Resources:**

- [ ] `KeyVault:VaultUri` - Azure Key Vault URI
- [ ] `KeyVault:UseManagedIdentity` - Set to true in production
- [ ] Application Insights configured (connection string from Key Vault or empty for MI)

---

## 19. Appendix

### A. Glossary

- **Entra ID**: Microsoft Entra ID (formerly Azure Active Directory)
- **App Registration**: Azure AD application registration for OAuth/OIDC
- **Graph API**: Microsoft Graph API for accessing Microsoft 365 data
- **Aspire**: .NET Aspire framework for cloud-native applications
- **Managed Identity (MI)**: Azure-managed identity for secure resource access without storing credentials
  - **System-Assigned MI**: Tied to the lifecycle of the Azure resource (preferred)
  - **User-Assigned MI**: Standalone identity that can be assigned to multiple resources
- **DefaultAzureCredential**: Azure SDK credential chain that automatically selects the best authentication method
- **TokenCredential**: Base type for all Azure SDK authentication mechanisms
- **OAuth 2.0**: Open standard for access delegation and authorization
- **PKCE**: Proof Key for Code Exchange - security extension for OAuth2
- **Delegated Permissions**: Permissions that require a signed-in user (user context)
- **Application Permissions**: Permissions for apps running without a user (app context)
- **Service Principal**: Enterprise application representation of an app or managed identity in Entra ID
- **Security Group**: Entra ID group used for access control and permissions
- **Group Claims**: User's group memberships included in authentication tokens
- **MSAL**: Microsoft Authentication Library for acquiring tokens
- **Azure RBAC**: Role-Based Access Control for Azure resource authorization
- **Key Vault**: Azure service for securely storing secrets, keys, and certificates

### B. References

**Core Technologies:**

- [Microsoft Graph API Documentation](https://learn.microsoft.com/graph)
- [.NET Aspire Documentation](https://learn.microsoft.com/dotnet/aspire)
- [Azure App Service Documentation](https://learn.microsoft.com/azure/app-service)

**Authentication & Authorization:**

- [Microsoft Identity Platform](https://learn.microsoft.com/entra/identity-platform)
- [Microsoft.Identity.Web Documentation](https://learn.microsoft.com/entra/msal/dotnet/microsoft-identity-web/)
- [OAuth 2.0 Authorization Code Flow](https://learn.microsoft.com/entra/identity-platform/v2-oauth2-auth-code-flow)
- [Configure Group Claims](https://learn.microsoft.com/entra/identity-platform/optional-claims)
- [Entra ID Security Groups](https://learn.microsoft.com/entra/fundamentals/how-to-manage-groups)

**Managed Identity & Azure SDK:**

- [Azure Identity SDK (Azure.Identity)](https://learn.microsoft.com/dotnet/api/overview/azure/identity-readme)
- [DefaultAzureCredential](https://learn.microsoft.com/dotnet/api/azure.identity.defaultazurecredential)
- [Managed Identities Overview](https://learn.microsoft.com/entra/identity/managed-identities-azure-resources/overview)
- [How to use Managed Identity with App Service](https://learn.microsoft.com/azure/app-service/overview-managed-identity)
- [Authenticate to Azure resources from .NET apps](https://learn.microsoft.com/dotnet/azure/sdk/authentication)
- [Best Practices for Azure SDK](https://learn.microsoft.com/dotnet/azure/sdk/best-practices)
- [Grant Managed Identity access to Microsoft Graph](https://learn.microsoft.com/graph/sdks/choose-authentication-providers#application-authentication)

**Azure Services:**

- [Azure Key Vault SDK](https://learn.microsoft.com/dotnet/api/overview/azure/security.keyvault.secrets-readme)
- [Azure Key Vault with Managed Identity](https://learn.microsoft.com/azure/key-vault/general/managed-identity)
- [Application Insights SDK](https://learn.microsoft.com/azure/azure-monitor/app/asp-net-core)
- [Azure Cache for Redis](https://learn.microsoft.com/azure/azure-cache-for-redis/cache-dotnet-core-quickstart)

### C. Contact & Support

- Project Owner: [Name]
- Technical Lead: [Name]
- Repository: [GitHub URL]

---

## 20. Implementation Summary - Managed Identity & Developer Experience

### Key Features

This application implements **Azure Managed Identity with DefaultAzureCredential** and provides an **exceptional developer experience**:

✅ **Zero Secrets in Production** - User-Assigned Managed Identity for all Azure resources  
✅ **Same Code Everywhere** - No conditional logic for local vs Azure  
✅ **Production-Like Permissions Locally** - Development Service Principal has same permissions as Production MI  
✅ **F5 Debugging** - Set credentials in User Secrets and press F5  
✅ **User Secrets** - Sensitive data in secrets.json, never in git  
✅ **DefaultAzureCredential** - Automatically picks right auth method  
✅ **Easy Onboarding** - Developers productive in < 5 minutes  
✅ **No Personal Admin Permissions Needed** - Developers use shared Service Principal

#### 1. **Credential Flow (DefaultAzureCredential Chain)**

```
Development (Local):
  Developer sets AZURE_CLIENT_ID, AZURE_CLIENT_SECRET, AZURE_TENANT_ID in User Secrets
    ↓
  Program.cs maps secrets to environment variables
    ↓
  DefaultAzureCredential uses EnvironmentCredential (checks env vars first!)
    ↓
  Authenticates as Development Service Principal
    ↓
  Service Principal has same Graph API permissions as Production MI
    ↓
  Application works exactly like production (same permissions!)
    ↓
  No personal admin permissions needed for developers!

Production (Azure):
  User-Assigned Managed Identity created and assigned to App Service
    ↓
  ManagedIdentity:ClientId configured in app settings
    ↓
  DefaultAzureCredential uses ManagedIdentityCredential (with Client ID)
    ↓
  Application authenticates as the User-Assigned Managed Identity
    ↓
  Accesses Azure resources with MI's assigned permissions
```

#### 2. **Local Development Experience (< 5 minutes to start)**

```bash
# Step 1: Install prerequisites (one-time)
# - .NET 10 SDK
# - Get Development Service Principal credentials from IT Admin

# Step 2: Clone repository (one-time)
git clone <repo-url>
cd entra-id-app-portal

# Step 3: Set secrets (one-time)
cd entra-id-app-portal.Web
dotnet user-secrets init

# Set Development Service Principal credentials (acts as Development MI)
dotnet user-secrets set "AZURE_CLIENT_ID" "<dev-sp-client-id>"
dotnet user-secrets set "AZURE_CLIENT_SECRET" "<dev-sp-client-secret>"
dotnet user-secrets set "AZURE_TENANT_ID" "<tenant-id>"
dotnet user-secrets set "Authorization:AdminGroupId" "<group-id>"
dotnet user-secrets set "Authorization:SupportGroupId" "<group-id>"

# Step 4: Press F5 - IT JUST WORKS! 🎉
# - Configuration loads from secrets.json
# - DefaultAzureCredential uses Service Principal (via EnvironmentCredential)
# - Service Principal has same permissions as Production MI
# - Same code runs in Azure (no changes!)
```

**Configuration Loading (Automatic):**
```
Local Development:
  1. secrets.json (User Secrets) ← Sensitive values
  2. local.settings.json (if present) ← Local overrides
  3. appsettings.Development.json ← Development settings
  4. appsettings.json ← Base configuration
  5. Environment variables ← Fallback

Azure Production:
  1. Azure Key Vault (via Managed Identity) ← Secrets
  2. App Service Configuration ← App Settings
  3. appsettings.json ← Base configuration
  4. Environment variables ← Fallback
```

**Same Code, Different Credentials:**
- 🏠 **Local**: Development Service Principal (via EnvironmentCredential) → Same permissions as Production MI
- ☁️ **Azure**: User-Assigned Managed Identity → Production permissions
- 🚀 **Zero code changes** between environments!
- ✅ **Both use application permissions** (not delegated) - true production parity!

#### 3. **Resource Access Pattern**

All Azure services are accessed using the same pattern (works locally AND in Azure):

```csharp
// Single credential instance for all services
var credential = new DefaultAzureCredential(new DefaultAzureCredentialOptions
{
    ManagedIdentityClientId = config["ManagedIdentity:ClientId"]  // Used in Azure, ignored locally
});

// Key Vault
var secretClient = new SecretClient(vaultUri, credential);

// Microsoft Graph
var graphClient = new GraphServiceClient(credential, scopes);

// Application Insights (automatic with MI)
builder.Services.AddApplicationInsightsTelemetry();

// No environment detection needed - it just works!
```

#### 4. **Zero-Secrets Checklist**

**Production (Azure):**
✅ **NO** `ClientSecret` in production configuration  
✅ **NO** connection strings with passwords  
✅ **NO** API keys in environment variables  
✅ **NO** SAS tokens hardcoded  
✅ All credentials acquired automatically by Azure  
✅ Production uses User-Assigned Managed Identity exclusively

**Development (Local):**
✅ Credentials in `secrets.json` (User Secrets) - **NOT in git**  
✅ Developers use **Development Service Principal** (shared, not personal)  
✅ Service Principal has **same permissions as Production MI**  
✅ Same configuration structure as production  
✅ Easy to debug and test locally  
✅ No personal admin permissions needed  
✅ No environment detection code needed

#### 5. **Required NuGet Packages**

```xml
<!-- Core Azure Identity -->
<PackageReference Include="Azure.Identity" Version="1.13.*" />

<!-- Azure Services -->
<PackageReference Include="Azure.Security.KeyVault.Secrets" Version="4.6.*" />
<PackageReference Include="Azure.Extensions.AspNetCore.Configuration.Secrets" Version="1.3.*" />

<!-- Microsoft Graph with Identity -->
<PackageReference Include="Microsoft.Graph" Version="5.*" />
<PackageReference Include="Microsoft.Identity.Web" Version="3.*" />
<PackageReference Include="Microsoft.Identity.Web.MicrosoftGraph" Version="3.*" />
```

#### 6. **User-Assigned Managed Identity Setup Commands**

```bash
# 1. Create User-Assigned Managed Identity
az identity create \
  --name entra-portal-identity \
  --resource-group <rg-name> \
  --location <location>

# 2. Get the IDs
CLIENT_ID=$(az identity show --name entra-portal-identity --resource-group <rg-name> --query clientId -o tsv)
PRINCIPAL_ID=$(az identity show --name entra-portal-identity --resource-group <rg-name> --query principalId -o tsv)
MI_RESOURCE_ID=$(az identity show --name entra-portal-identity --resource-group <rg-name> --query id -o tsv)

echo "Client ID (for config): $CLIENT_ID"
echo "Principal ID (for permissions): $PRINCIPAL_ID"

# 3. Grant Key Vault access
az role assignment create \
  --role "Key Vault Secrets User" \
  --assignee $PRINCIPAL_ID \
  --scope /subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.KeyVault/vaults/<kv-name>

# 4. Assign to App Service
az webapp identity assign \
  --name <app-name> \
  --resource-group <rg-name> \
  --identities $MI_RESOURCE_ID

# 5. Configure Client ID in App Settings
az webapp config appsettings set \
  --name <app-name> \
  --resource-group <rg-name> \
  --settings ManagedIdentity__ClientId=$CLIENT_ID ManagedIdentity__Enabled=true
```

```powershell
# Grant Graph API permissions to User-Assigned Managed Identity
Connect-MgGraph -Scopes "Application.ReadWrite.All", "AppRoleAssignment.ReadWrite.All"

# Replace with your User-Assigned MI Principal ID
$managedIdentityPrincipalId = "<user-assigned-mi-principal-id>"
$graphSp = Get-MgServicePrincipal -Filter "displayName eq 'Microsoft Graph'"

# Grant multiple permissions
$permissions = @("Application.Read.All", "Directory.Read.All", "GroupMember.Read.All")

foreach ($permission in $permissions) {
    $appRole = $graphSp.AppRoles | Where-Object { $_.Value -eq $permission }
    New-MgServicePrincipalAppRoleAssignment `
      -ServicePrincipalId $managedIdentityPrincipalId `
      -PrincipalId $managedIdentityPrincipalId `
      -ResourceId $graphSp.Id `
      -AppRoleId $appRole.Id
    Write-Host "✅ Granted $permission"
}
```

#### 7. **Benefits Achieved**

| Benefit                      | Description                                 |
| ---------------------------- | ------------------------------------------- |
| 🔐 **Zero Secrets**          | No credentials to manage, rotate, or leak   |
| 🛡️ **Enhanced Security**     | Azure manages all credentials automatically |
| 🚀 **Simplified Deployment** | No secret injection in CI/CD pipelines      |
| 📊 **Complete Audit Trail**  | All access logged with MI identity          |
| 👨‍💻 **Developer Friendly**    | Seamless local development with `az login`  |
| ♻️ **No Rotation Required**  | Azure handles credential lifecycle          |
| 🎯 **Least Privilege**       | Fine-grained RBAC permissions per resource  |

#### 8. **Authentication Architecture**

```
┌─────────────────────────────────────────────────────────────┐
│                       User Access                            │
│  (OAuth2 Code Flow - Microsoft.Identity.Web)                │
│                                                              │
│  User → Azure AD Login → ID Token → Portal Access           │
│                                                              │
│  Groups: Entra-Admin (full) or Entra-Support (read-only)   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                  Blazor Web Application                      │
│                (User Context - Delegated)                    │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│                     Backend API Service                      │
│            (App Context - Managed Identity)                  │
│                                                              │
│  DefaultAzureCredential → ManagedIdentityCredential          │
└─────────────┬───────────────────────────────┬───────────────┘
              │                               │
              ▼                               ▼
    ┌─────────────────┐           ┌──────────────────┐
    │ Microsoft Graph │           │  Azure Key Vault │
    │  (App Perms)    │           │   (RBAC Access)  │
    │                 │           │                  │
    │ - Get Apps      │           │ - Group IDs      │
    │ - Delete Apps   │           │ - Config Values  │
    └─────────────────┘           └──────────────────┘
```

#### 9. **Troubleshooting Quick Reference**

| Issue                                     | Solution                                         |
| ----------------------------------------- | ------------------------------------------------ |
| Local: "Failed to retrieve token"         | Run `az login` and verify with `az account show` |
| Azure: "ManagedIdentityCredential failed" | Enable System-Assigned MI on App Service         |
| "Access denied" to Key Vault              | Grant "Key Vault Secrets User" role to MI        |
| "AADSTS700016" Graph API error            | Grant Graph API app roles to MI using PowerShell |
| Token acquisition slow                    | Check MI is enabled and permissions are correct  |

#### 10. **Critical Deployment Steps**

**Pre-Deployment (Create Identity & Configure Permissions):**
1. ✅ Create User-Assigned Managed Identity (`entra-portal-identity`)
2. ✅ Document Client ID (for app config) and Principal ID (for permissions)
3. ✅ Grant MI access to Key Vault (RBAC: "Key Vault Secrets User")
4. ✅ Grant MI access to Microsoft Graph (PowerShell: App Role Assignment)
   - Application.Read.All (or Application.ReadWrite.All)
   - Directory.Read.All
   - GroupMember.Read.All
5. ✅ Grant MI access to Application Insights (if needed)

**Deployment Configuration:**
6. ✅ Update configuration: `ManagedIdentity:Enabled = true`
7. ✅ Update configuration: `ManagedIdentity:ClientId = <user-assigned-mi-client-id>`
8. ✅ Update configuration: `ManagedIdentity:UseManagedIdentityForGraph = true`
9. ✅ Remove any client secrets from production configuration
10. ✅ Verify `DefaultAzureCredential` includes `ManagedIdentityClientId` in options

**App Service Setup:**
11. ✅ Assign User-Assigned MI to App Service (Azure Portal or CLI)
12. ✅ Configure App Settings with MI Client ID
13. ✅ Test token acquisition in staging environment first

**Verification:**
14. ✅ Verify MI can acquire token for Graph API
15. ✅ Verify all Graph API calls work with MI
16. ✅ Verify Key Vault access works with MI
17. ✅ Test application end-to-end in staging
18. ✅ Deploy to production

---

**Document Version**: 2.3 (User-Assigned MI + Development Service Principal)  
**Last Updated**: November 22, 2025  
**Status**: Ready for Development  
**Security Model**: Zero-Secrets Architecture with User-Assigned Managed Identity  
**Developer Experience**: Development Service Principal provides production-like permissions locally - Set credentials in User Secrets and F5!  
**Key Innovation**: Developers don't need admin permissions - Use shared Development SP with MI-like permissions
