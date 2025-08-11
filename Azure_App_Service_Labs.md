# Azure App Service Labs

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Exercise 1 – Create an App Service in Azure](#exercise-1--create-an-app-service-in-azure)
3. [Exercise 2 – Create ASP.NET Core 8.0 Application](#exercise-2--create-aspnet-core-80-application)
4. [Exercise 3 – Deploy Your First App to Azure App Service using VS Code](#exercise-3--deploy-your-first-app-to-azure-app-service-using-vs-code)
5. [Exercise 4 – Create a Basic Notes Application with SQL Database Integration](#exercise-4--create-a-basic-notes-application-with-sql-database-integration)
6. [Exercise 5 – Razor Page UI and Backend Logic](#exercise-5--razor-page-ui-and-backend-logic)
7. [Step 7 – Configure Local App Settings (User Secrets)](#step-7--configure-local-app-settings-user-secrets)
8. [Step 8 – Configure Azure App Service Settings](#step-8--configure-azure-app-service-settings)
9. [Step 9 – Deploy to Azure App Service](#step-9--deploy-to-azure-app-service)
10. [Lab 03 – Application Insights Integration](#lab-03--application-insights-integration)
11. [Testing Telemetry](#testing-telemetry)
12. [Deploy to Slots and Switch Slots](#deploy-to-slots-and-switch-slots)

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

## Exercise 3 – Deploy Your First App to Azure App Service using VS Code

1. Open your project in VS Code.
2. Sign in to Azure (`Ctrl+Shift+P` → **Azure: Sign In**).
3. `Ctrl+Shift+P` → **Azure App Service: Deploy to Web App**.
4. Select your subscription and App Service.
5. Confirm **Deploy**.
6. Open the deployed site from the notification.

---

## Exercise 4 – Create a Basic Notes Application with SQL Database Integration

Includes:
- Install `Microsoft.EntityFrameworkCore.SqlServer`
- Create `Models/GuestMessage.cs`
- Create `Data/AppDbContext.cs`
- Update `Program.cs` with EF Core and connection string registration.

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

**GuestMessage.cs**
```csharp
// Full model code here...
```

**AppDbContext.cs**
```csharp
// Full context code here...
```

**Program.cs changes**
```csharp
// EF Core registration snippet here...
```

---

## Exercise 5 – Razor Page UI and Backend Logic

**Index.cshtml**
```cshtml
// Full Razor Page markup...
```

**Index.cshtml.cs**
```csharp
// Full backend logic code...
```

---

## Step 7 – Configure Local App Settings (User Secrets)

```bash
cd MyFirstAzureWebApp
dotnet user-secrets init
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=tcp:<server>.database.windows.net,..."
```

---

## Step 8 – Configure Azure App Service Settings

- **SQL Firewall:** Enable *Allow Azure services* or add outbound IPs.
- **App Service → Configuration → Connection strings:**
  - Name: `DefaultConnection`
  - Type: `SQLAzure`
  - Value: your connection string.

---

## Step 9 – Deploy to Azure App Service

```bash
dotnet publish -c Release -o ./publish
```

Deploy via VS Code:
- Right-click `publish` folder → **Deploy to Web App…**

Test:
```
https://<app>.azurewebsites.net/Notes
```

---

## Lab 03 – Application Insights Integration

### Create Application Insights
- Azure Portal → Create resource → Application Insights.
- Copy **Connection String**.

### Add AI to the App
```bash
dotnet add package Microsoft.ApplicationInsights.AspNetCore
```

**Program.cs changes**
```csharp
// App Insights setup code...
```

**Index.cshtml.cs changes**
```csharp
// TelemetryClient injection and TrackEvent usage...
```

### Configure Connection Strings
```bash
dotnet user-secrets set "ApplicationInsights:ConnectionString" "<your-connection-string>"
```

Set in Azure App Service → Configuration.

---

## Testing Telemetry

- Generate traffic by browsing `/Notes`.
- View data in Azure Portal:
  - Live Metrics
  - Transaction Search
  - Application Map
- Use Kusto queries for requests, dependencies, events.

---

## Deploy to Slots and Switch Slots

1. Make a visible change (e.g., add **STAGING VERSION** text).
2. Azure Portal → App Service → Deployment slots → Add Slot → `stage`.
3. Deploy to `stage` slot via VS Code.
4. Verify staging URL.
5. Swap slot with production.

> ✅ Slots allow zero-downtime swaps and quick rollbacks.

---

**End of Labs**
