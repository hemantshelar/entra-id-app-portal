# Entra ID App Registration Portal - Technical Requirements Document

## 1. Project Overview

### 1.1 Purpose

Build a modern web portal to manage and monitor Azure Entra ID (formerly Azure AD) application registrations with filtering, monitoring, and management capabilities.

### 1.2 Goals

- Provide a user-friendly interface to view and manage Entra ID app registrations
- Enable proactive monitoring of expiring secrets and certificates
- Reduce Microsoft Graph API calls through intelligent caching
- Deploy to Azure cloud with scalability and security best practices

### 1.3 Target Audience

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
- **Caching**: IMemoryCache or IDistributedCache (Redis for production)

### 2.4 Architecture Patterns

- **Dependency Injection (DI)**: Built-in ASP.NET Core DI container
- **Repository Pattern**: For data access abstraction
- **Service Layer Pattern**: Business logic separation
- **CQRS Pattern**: For command/query separation (if complexity requires)
- **Testing**: BDD with SpecFlow or xUnit with FluentAssertions

### 2.5 Cloud & DevOps

- **Hosting**: Azure App Service or Azure Container Apps
- **Configuration**: Azure Key Vault for secrets
- **Monitoring**: Application Insights
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
│   │   └── CacheService.cs               # Cache management
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

1. **User Secrets** (`secrets.json`) - Development only
2. **Local Settings** (`appsettings.Development.json`) - Development
3. **Azure Key Vault** - Production
4. **Environment Variables** - Fallback

### 6.2 Required Configuration Keys

```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "<tenant-id>",
    "ClientId": "<client-id>",
    "ClientSecret": "<client-secret>", // Vault or secrets.json only
    "CallbackPath": "/signin-oidc",
    "Scopes": "https://graph.microsoft.com/.default"
  },
  "GraphApi": {
    "BaseUrl": "https://graph.microsoft.com/v1.0",
    "BatchSize": 999,
    "RetryAttempts": 3,
    "RetryDelaySeconds": 2
  },
  "Cache": {
    "ExpirationMinutes": 60,
    "EnableAutoRefresh": false,
    "AutoRefreshIntervalMinutes": 30
  },
  "Features": {
    "EnableDelete": true,
    "EnableExport": true,
    "EnableBulkOperations": false
  },
  "Monitoring": {
    "ApplicationInsights": {
      "ConnectionString": "<connection-string>"
    }
  }
}
```

### 6.3 Configuration Implementation

```csharp
// Program.cs - Configuration loading order
var builder = WebApplication.CreateBuilder(args);

// 1. Base configuration (appsettings.json)
// Already loaded by default

// 2. Environment-specific configuration
// Already loaded by default

// 3. User Secrets (development only)
if (builder.Environment.IsDevelopment())
{
    builder.Configuration.AddUserSecrets<Program>();
}

// 4. Environment variables (always)
builder.Configuration.AddEnvironmentVariables();

// 5. Azure Key Vault (production)
if (!builder.Environment.IsDevelopment())
{
    var keyVaultUrl = builder.Configuration["KeyVault:VaultUri"];
    if (!string.IsNullOrEmpty(keyVaultUrl))
    {
        builder.Configuration.AddAzureKeyVault(
            new Uri(keyVaultUrl),
            new DefaultAzureCredential());
    }
}

// Options pattern
builder.Services.Configure<AzureAdOptions>(
    builder.Configuration.GetSection("AzureAd"));
builder.Services.Configure<GraphApiOptions>(
    builder.Configuration.GetSection("GraphApi"));
```

---

## 7. Authentication & Authorization

### 7.1 Authentication Flow

- Use Microsoft Identity Platform (MSAL)
- Implement OAuth 2.0 authorization code flow
- Acquire tokens for Microsoft Graph API
- Support for token refresh

### 7.2 Required App Registration Permissions

**Microsoft Graph API Permissions** (Application type):

- `Application.Read.All` (minimum for read operations)
- `Application.ReadWrite.All` (required for delete operations)
- `Directory.Read.All` (for owner information)

**Delegated Permissions** (if using delegated auth):

- `User.Read`
- `Application.Read.All`

### 7.3 Authorization Roles

| Role              | Permissions                                |
| ----------------- | ------------------------------------------ |
| **Reader**        | View all app registrations, filter, export |
| **Contributor**   | Reader + Refresh cache                     |
| **Administrator** | Contributor + Delete app registrations     |

Implementation:

```csharp
[Authorize(Roles = "Administrator")]
[HttpDelete("{id}")]
public async Task<IActionResult> DeleteAppRegistration(string id)
{
    // Delete logic
}
```

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

- **Header**: App title, user info, logout button
- **Navigation**: Side menu or top nav with sections:
  - Dashboard
  - All Applications
  - Expiring Secrets
  - Settings (future)
- **Main Content**: Data grid/table with filters panel
- **Footer**: Copyright, version, last refresh time

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
- Authentication flow
- API endpoints (end-to-end)

**Framework**: WebApplicationFactory + xUnit

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

| Resource                 | Purpose                                | SKU/Tier                    |
| ------------------------ | -------------------------------------- | --------------------------- |
| Azure App Service        | Host web app and API                   | B1 (Basic) or S1 (Standard) |
| Azure Key Vault          | Store secrets                          | Standard                    |
| Azure Cache for Redis    | Distributed cache (optional)           | Basic C0                    |
| Application Insights     | Monitoring and telemetry               | Pay-as-you-go               |
| Azure Container Registry | Container images (if using containers) | Basic                       |

### 12.2 Deployment Options

**Option 1: Azure App Service (Direct Deploy)**

- Build and publish from CI/CD pipeline
- Deploy as .NET application
- Use App Service configuration for settings

**Option 2: Container-based (Preferred for Aspire)**

- Build Docker images via Aspire
- Push to Azure Container Registry
- Deploy to Azure Container Apps or AKS

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

- Use User Secrets for local testing
- Point to development tenant

**Staging**:

- Use Azure Key Vault
- Separate test Entra ID tenant (or test apps)

**Production**:

- Use Azure Key Vault
- Managed Identity for authentication (no secrets in config)
- Production tenant

---

## 13. Security Best Practices

1. **Authentication**:

   - Always use HTTPS
   - Implement proper token refresh
   - Use Managed Identity in Azure (no secrets)

2. **Authorization**:

   - Implement RBAC
   - Least privilege principle
   - Audit logs for sensitive operations

3. **Data Protection**:

   - Never log sensitive data
   - Encrypt data in transit (TLS 1.2+)
   - Use Azure Key Vault for secrets

4. **API Security**:

   - Implement rate limiting
   - CORS policy (whitelist origins)
   - Content Security Policy headers
   - Anti-forgery tokens

5. **Dependency Management**:
   - Keep NuGet packages up-to-date
   - Monitor for security vulnerabilities
   - Use Dependabot for automated updates

---

## 14. Development Guidelines

### 14.1 Code Standards

- Follow Microsoft C# coding conventions
- Use StyleCop for code analysis
- XML documentation for public APIs
- Meaningful names (no abbreviations)

### 14.2 Git Workflow

- Feature branches: `feature/description`
- Bug fixes: `bugfix/description`
- Pull requests required for main branch
- Commit messages: Use conventional commits

### 14.3 Code Review Checklist

- [ ] Code follows style guidelines
- [ ] Unit tests added/updated
- [ ] No hardcoded secrets
- [ ] Error handling implemented
- [ ] Logging added for important operations
- [ ] XML comments added
- [ ] No console warnings or errors

### 14.4 Performance Considerations

- Use async/await properly
- Avoid N+1 queries
- Implement pagination for large datasets
- Cache frequently accessed data
- Use connection pooling

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

### Launch Checklist:

- [ ] Production Entra ID app registration created
- [ ] Azure resources provisioned
- [ ] Key Vault configured with secrets
- [ ] CI/CD pipeline configured
- [ ] Monitoring and alerts configured
- [ ] User roles assigned
- [ ] Backup and disaster recovery plan
- [ ] Runbook for common operations

---

## 18. Appendix

### A. Glossary

- **Entra ID**: Microsoft Entra ID (formerly Azure Active Directory)
- **App Registration**: Azure AD application registration for OAuth/OIDC
- **Graph API**: Microsoft Graph API for accessing Microsoft 365 data
- **Aspire**: .NET Aspire framework for cloud-native applications
- **Managed Identity**: Azure AD identity for Azure resources

### B. References

- [Microsoft Graph API Documentation](https://learn.microsoft.com/graph)
- [.NET Aspire Documentation](https://learn.microsoft.com/dotnet/aspire)
- [Azure App Service Documentation](https://learn.microsoft.com/azure/app-service)
- [Microsoft Identity Platform](https://learn.microsoft.com/entra/identity-platform)

### C. Contact & Support

- Project Owner: [Name]
- Technical Lead: [Name]
- Repository: [GitHub URL]

---

**Document Version**: 1.0  
**Last Updated**: November 22, 2025  
**Status**: Ready for Development
