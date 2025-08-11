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
// TODO: Add full model code here
```

### Create `Data/AppDbContext.cs`
```csharp
// TODO: Add DbContext code here
```

### Update `Program.cs`
```csharp
// TODO: Add EF Core registration code here
```

### Create Razor Page `Pages/Notes/Index.cshtml`
```cshtml
// TODO: Add Razor markup here
```

### Create Page Model `Pages/Notes/Index.cshtml.cs`
```csharp
// TODO: Add backend logic code here
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
// TODO: Add AI setup code
```

**Index.cshtml.cs changes**
```csharp
// TODO: Add TelemetryClient usage
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
