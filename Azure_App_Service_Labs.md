# Azure App Service Labs

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Exercise 1 – Create an App Service in Azure](#exercise-1--create-an-app-service-in-azure)
3. [Exercise 2 – Create ASP.NET Core 8.0 Application](#exercise-2--create-aspnet-core-80-application)
4. [Exercise 3 – Deploy Your First Application to Azure App Service](#exercise-3--deploy-your-first-application-to-azure-app-service)
5. [Exercise 4 – Create a Basic Notes Application with SQL Database Integration](#exercise-4--create-a-basic-notes-application-with-sql-database-integration)
6. [Exercise 5 – Integrate Application Insights](#exercise-5--integrate-application-insights)
7. [Exercise 6 – Deploy to Slots and Swap Slots](#exercise-6--deploy-to-slots-and-swap-slots)

---

## Prerequisites

Before starting this lab, ensure the following tools are installed and set up on your machine:

1. **.NET 8 SDK**  
   - Download: [https://dotnet.microsoft.com/en-us/download/dotnet/8.0](https://dotnet.microsoft.com/en-us/download/dotnet/8.0)  
   - Verify installation:  
     ```bash
     dotnet --version
     ```

2. **Visual Studio Code**  
   - Download: [https://code.visualstudio.com/](https://code.visualstudio.com/)  
   - Install the **C# Dev Kit** extension from the Extensions Marketplace.

3. **Azure CLI**  
   - Download: [https://learn.microsoft.com/en-us/cli/azure/install-azure-cli](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)  
   - Verify installation:  
     ```bash
     az --version
     ```

4. **Azure App Service Extension for VS Code**  
   - In VS Code, go to **Extensions** (`Ctrl+Shift+X` / `Cmd+Shift+X` on Mac).  
   - Search for **Azure App Service** and install.

> 💡 **Tip:** Sign in to your Azure account in **both** the Azure CLI and VS Code before starting.

---

## Exercise 1 – Create an App Service in Azure

Follow your instructor for this exercise.

---

## Exercise 2 – Create ASP.NET Core 8.0 Application

```bash
dotnet new webapp -n MyFirstAzureWebApp --framework net8.0
cd MyFirstAzureWebApp
dotnet run --urls=https://localhost:5001/
```

- Open browser: [https://localhost:5001/](https://localhost:5001/)  

Publish:
```bash
dotnet publish -c Release -o ./publish
```

---

## Exercise 3 – Deploy Your First Application to Azure App Service

1. Open your project in VS Code.
2. Sign in to Azure (`Ctrl+Shift+P` → **Azure: Sign In**).
3. `Ctrl+Shift+P` → **Azure App Service: Deploy to Web App**.
4. Select your subscription and App Service.
5. Confirm **Deploy**.
6. Open the deployed site from the notification.

---

## Exercise 4 – Create a Basic Notes Application with SQL Database Integration

This exercise covers:
- Installing EF Core SQL Server provider
- Creating the data model
- Creating the DbContext
- Wiring up EF Core and connection strings
- Adding Razor Pages for UI
- Configuring local settings securely
- Configuring Azure App Service settings
- Deploying to Azure

### Install SQL Server provider
```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

### Create `Models/GuestMessage.cs`
```csharp
using System;
using System.ComponentModel.DataAnnotations;
namespace MyFirstAzureWebApp.Models
{
    public class GuestMessage
    {
        public int Id { get; set; }
[Required, StringLength(100)]
        public string UserName { get; set; } = string.Empty;
[Required, StringLength(500)]
        public string Message { get; set; } = string.Empty;
public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    }
}

```

### Create `Data/AppDbContext.cs`
```csharp
using Microsoft.EntityFrameworkCore;
using MyFirstAzureWebApp.Models;
namespace MyFirstAzureWebApp.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
        public DbSet<GuestMessage> Messages => Set<GuestMessage>();
    }
}
```

### Update `Program.cs`
```csharp
using Microsoft.EntityFrameworkCore;                 // <-- add
using MyFirstAzureWebApp.Data;                      // <-- add
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddRazorPages();
// Read "DefaultConnection" from config (user-secrets locally, App Service in cloud)
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
var app = builder.Build();
if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Error");
    app.UseHsts();
}
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();
app.MapRazorPages();
app.Run();

```

### Create Razor Page `Pages/Notes/Index.cshtml`
```cshtml
@page
@model MyFirstAzureWebApp.Pages.Notes.IndexModel
@{
    ViewData["Title"] = "Notes";
}

<h2>Notes</h2>

<form method="post">
    <div class="mb-3">
        <label asp-for="Input.UserName" class="form-label"></label>
        <input asp-for="Input.UserName" class="form-control" />
        <span asp-validation-for="Input.UserName" class="text-danger"></span>
    </div>

    <div class="mb-3">
        <label asp-for="Input.Message" class="form-label"></label>
        <textarea asp-for="Input.Message" class="form-control"></textarea>
        <span asp-validation-for="Input.Message" class="text-danger"></span>
    </div>

    <!-- Add validates; others skip validation -->
    <button type="submit" asp-page-handler="Add" class="btn btn-primary">Add</button>
    <button type="submit" asp-page-handler="DeleteByUser" class="btn btn-danger ms-2" formnovalidate
            title="Deletes all messages for the given username (Message can be empty)">Delete by user</button>
    <button type="submit" asp-page-handler="Show" class="btn btn-outline-secondary ms-2" formnovalidate>Show messages</button>
    <button type="submit" asp-page-handler="Hide" class="btn btn-outline-secondary ms-2" formnovalidate>Hide</button>
</form>

<hr />

@if (Model.ShowList && Model.Items?.Any() == true)
{
    <ul class="list-group">
        @foreach (var m in Model.Items)
        {
            <li class="list-group-item">
                <strong>@m.UserName</strong>: @m.Message
                <small class="text-muted">(@m.CreatedAt.ToLocalTime())</small>
            </li>
        }
    </ul>
}
else if (Model.ShowList)
{
    <p>No entries yet.</p>
}

```

### Create Page Model `Pages/Notes/Index.cshtml.cs`
```csharp
  using Microsoft.AspNetCore.Mvc;
  using Microsoft.AspNetCore.Mvc.RazorPages;
  using Microsoft.EntityFrameworkCore;
  using System.ComponentModel.DataAnnotations;
  using MyFirstAzureWebApp.Data;
  using MyFirstAzureWebApp.Models;
  
  namespace MyFirstAzureWebApp.Pages.Notes
  {
      public class IndexModel : PageModel
      {
          private readonly AppDbContext _db;
          private readonly ILogger<IndexModel> _log;
          public IndexModel(AppDbContext db, ILogger<IndexModel> log) { _db = db; _log = log; }
  
          public List<GuestMessage> Items { get; set; } = new();
          public bool ShowList { get; set; } = true;
  
          [BindProperty]
          public InputModel Input { get; set; } = new();
  
          public async Task OnGetAsync()
          {
              Items = await _db.Messages.OrderByDescending(x => x.Id).ToListAsync();
              ShowList = true; // show by default
          }
  
          // Add requires both fields
          public async Task<IActionResult> OnPostAddAsync()
          {
              if (!ModelState.IsValid)
              {
                  Items = await _db.Messages.OrderByDescending(x => x.Id).ToListAsync();
                  ShowList = true;
                  return Page();
              }
  
              var msg = new GuestMessage { UserName = Input.UserName, Message = Input.Message };
              _db.Messages.Add(msg);
              await _db.SaveChangesAsync();
              _log.LogInformation("Saved message from {User}", Input.UserName);
  
              return RedirectToPage();
          }
  
          // Delete by username (message optional)
          public async Task<IActionResult> OnPostDeleteByUserAsync()
          {
              if (string.IsNullOrWhiteSpace(Input.UserName))
              {
                  ModelState.AddModelError("Input.UserName", "Username is required to delete.");
                  Items = await _db.Messages.OrderByDescending(x => x.Id).ToListAsync();
                  ShowList = true;
                  return Page();
              }
  
              var toDelete = _db.Messages.Where(m => m.UserName == Input.UserName);
              int count = await toDelete.CountAsync();
              _db.Messages.RemoveRange(toDelete);
              await _db.SaveChangesAsync();
              _log.LogInformation("Deleted {Count} messages for {User}", count, Input.UserName);
  
              return RedirectToPage();
          }
  
          public async Task<IActionResult> OnPostShowAsync()
          {
              Items = await _db.Messages.OrderByDescending(x => x.Id).ToListAsync();
              ShowList = true;
              return Page();
          }
  
          public IActionResult OnPostHideAsync()
          {
              ShowList = false;
              Items = new();
              return Page();
          }
  
          public class InputModel
          {
              [Required, StringLength(100)]
              public string UserName { get; set; } = string.Empty;
  
              [Required, StringLength(500)]
              public string Message { get; set; } = string.Empty;
          }
      }
  }

```

### Configure Local Settings (User Secrets)
```bash
cd MyFirstAzureWebApp
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=tcp:<server>.database.windows.net,..."
```

### Configure Azure App Service Settings
- **SQL Firewall:** Enable *Allow Azure services* or add outbound IPs.
- **App Service → Configuration → Connection strings:**
  - Name: `DefaultConnection`
  - Type: `SQLAzure`
  - Value: your connection string.

### Deploy the Notes App to Azure
```bash
dotnet publish -c Release -o ./publish
```
Deploy via VS Code → Right-click `publish` folder → **Deploy to Web App…**

---

## Exercise 5 – Integrate Application Insights

This exercise covers:
- Creating an Application Insights resource
- Adding AI SDK to the app
- Updating `Program.cs` for telemetry
- Injecting and using `TelemetryClient` in the Notes app
- Configuring connection strings locally and in Azure

### Create Application Insights
- Azure Portal → Create resource → Application Insights.
- Copy **Connection String**.

### Add AI to the App
```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

**Program.cs changes**
```csharp
  using Microsoft.EntityFrameworkCore;                 // <-- add
  using MyFirstAzureWebApp.Data; 
  using Microsoft.ApplicationInsights.DependencyCollector;                      // <-- add
  var builder = WebApplication.CreateBuilder(args);
  builder.Services.AddRazorPages();
  // Read "DefaultConnection" from config (user-secrets locally, App Service in cloud)
  builder.Services.AddDbContext<AppDbContext>(options =>
      options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
  // Enables auto-collection of requests, dependencies, exceptions, and ILogger traces
  builder.Services.AddApplicationInsightsTelemetry();
  // NEW: capture SQL command text in dependencies.data
  builder.Services.ConfigureTelemetryModule<DependencyTrackingTelemetryModule>((m, _) =>
  {
      m.EnableSqlCommandTextInstrumentation = true;
  });
  var app = builder.Build();
  if (!app.Environment.IsDevelopment())
  {
      app.UseExceptionHandler("/Error");
      app.UseHsts();
  }
  app.UseHttpsRedirection();
  app.UseStaticFiles();
  app.UseRouting();
  app.UseAuthorization();
  app.MapRazorPages();
  app.Run();

```

**Index.cshtml.cs changes**
```csharp
  using Microsoft.AspNetCore.Mvc;
  using Microsoft.AspNetCore.Mvc.RazorPages;
  using Microsoft.EntityFrameworkCore;
  using System.ComponentModel.DataAnnotations;
  using MyFirstAzureWebApp.Data;
  using MyFirstAzureWebApp.Models;
  using Microsoft.ApplicationInsights; // Added for telemetry
  
  namespace MyFirstAzureWebApp.Pages.Notes
  {
      public class IndexModel : PageModel
      {
          private readonly AppDbContext _db;
          private readonly ILogger<IndexModel> _log;
          private readonly TelemetryClient _telemetry; // Added
  
          public IndexModel(AppDbContext db, ILogger<IndexModel> log, TelemetryClient telemetry)
          {
              _db = db;
              _log = log;
              _telemetry = telemetry;
          }
  
          public List<GuestMessage> Items { get; set; } = new();
          public bool ShowList { get; set; } = true;
  
          [BindProperty]
          public InputModel Input { get; set; } = new();
  
          public async Task OnGetAsync()
          {
              Items = await _db.Messages.OrderByDescending(x => x.Id).ToListAsync();
              ShowList = true; // show by default
          }
  
          // Add requires both fields
          public async Task<IActionResult> OnPostAddAsync()
          {
              if (!ModelState.IsValid)
              {
                  Items = await _db.Messages.OrderByDescending(x => x.Id).ToListAsync();
                  ShowList = true;
                  return Page();
              }
  
              var msg = new GuestMessage { UserName = Input.UserName, Message = Input.Message };
              _db.Messages.Add(msg);
              await _db.SaveChangesAsync();
  
              _log.LogInformation("Saved message from {User}", Input.UserName);
  
              // Track custom event for adding a note
              _telemetry.TrackEvent("NoteAdded", new Dictionary<string, string?>
              {
                  ["UserName"] = Input.UserName
              });
  
              return RedirectToPage();
          }
  
          // Delete by username (message optional)
          public async Task<IActionResult> OnPostDeleteByUserAsync()
          {
              if (string.IsNullOrWhiteSpace(Input.UserName))
              {
                  ModelState.AddModelError("Input.UserName", "Username is required to delete.");
                  Items = await _db.Messages.OrderByDescending(x => x.Id).ToListAsync();
                  ShowList = true;
                  return Page();
              }
  
              var toDelete = _db.Messages.Where(m => m.UserName == Input.UserName);
              int count = await toDelete.CountAsync();
              _db.Messages.RemoveRange(toDelete);
              await _db.SaveChangesAsync();
  
              _log.LogInformation("Deleted {Count} messages for {User}", count, Input.UserName);
  
              // Track custom event + metric for deletion
              _telemetry.TrackEvent(
                  "NotesDeletedByUser",
                  new Dictionary<string, string?> { ["UserName"] = Input.UserName },
                  new Dictionary<string, double> { ["DeletedCount"] = count });
  
              return RedirectToPage();
          }
  
          public async Task<IActionResult> OnPostShowAsync()
          {
              Items = await _db.Messages.OrderByDescending(x => x.Id).ToListAsync();
              ShowList = true;
  
              // Track event with metric for showing notes
              _telemetry.TrackEvent("NotesShown", null, new Dictionary<string, double>
              {
                  ["ItemCount"] = Items.Count
              });
  
              return Page();
          }
  
          public IActionResult OnPostHideAsync()
          {
              ShowList = false;
              Items = new();
              return Page();
          }
  
          public class InputModel
          {
              [Required, StringLength(100)]
              public string UserName { get; set; } = string.Empty;
  
              [Required, StringLength(500)]
              public string Message { get; set; } = string.Empty;
          }
      }
}
```

### Configure Connection Strings
```bash
dotnet user-secrets set "ApplicationInsights:ConnectionString" "<your-connection-string>"
```

Set in Azure App Service → Configuration.

### Test Telemetry
- Generate traffic
- Use Live Metrics, Transaction Search, and Application Map in Azure Portal.

---

## Exercise 6 – Deploy to Slots and Swap Slots

1. Make a visible change (e.g., add **STAGING VERSION** text).
2. Azure Portal → App Service → Deployment slots → Add Slot → `stage`.
3. Deploy to `stage` slot via VS Code.
4. Verify staging URL.
5. Swap slot with production.

> ✅ Slots allow zero-downtime swaps and quick rollbacks.

---

**End of Labs**
