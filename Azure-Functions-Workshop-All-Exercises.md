# Azure Functions Workshop — Three Exercises

This workshop contains **three independent Azure Functions exercises**, each with a clear scenario and step‑by‑step guide. You can run them locally first and then deploy to Azure.

> **Runtime:** .NET 8 (Isolated) · Azure Functions v4  
> **Tools:** VS Code, Azure Functions Core Tools, Azure CLI  
> **Services used:** Azure Storage, Azure SQL, Azure Logic Apps, Azure AI Vision (Computer Vision)

---

## Table of Contents
- [Prerequisites](#prerequisites)
- [Exercise 1 — Contact Form → Azure SQL (HTTP Trigger)](#exercise-1--contact-form--azure-sql-http-trigger)
- [Exercise 2 — Image Upload → AI Tagging → Azure SQL (Blob Trigger)](#exercise-2--image-upload--ai-tagging--azure-sql-blob-trigger)
- [Exercise 3 — Nightly Image Summary + Cleanup + Email (Timer Trigger + Logic App)](#exercise-3--nightly-image-summary--cleanup--email-timer-trigger--logic-app)
- [Deploying to Azure](#deploying-to-azure)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)

---

## Prerequisites
- https://learn.microsoft.com/en-us/azure/azure-functions/functions-develop-vs-code?tabs=node-v4%2Cpython-v2%2Cisolated-process%2Cquick-create&pivots=programming-language-csharp#prerequisites


> **Important:** Do **not** commit `local.settings.json` to source control. Provide a `local.settings.sample.json` with placeholders instead.

---

## Exercise 1 — Contact Form → Azure SQL (HTTP Trigger)

### Scenario
Accept a form submission (name, email, message) via **HTTP POST** and save it to **Azure SQL**.

### Azure SQL: create table (Skip if you already have DB setup, or the instructor is providing the DB details)
Use the Azure Portal Query editor or your SQL tool of choice:
```sql
CREATE TABLE [dbo].[ContactMessages](
    [Id] INT IDENTITY(1,1) PRIMARY KEY,
    [Name] NVARCHAR(100),
    [Email] NVARCHAR(100),
    [Message] NVARCHAR(MAX),
    [CreatedAt] DATETIME DEFAULT GETDATE()
);
```

### Create the Function locally
```bash
func init ContactFunctionApp --worker-runtime dotnet-isolated --target-framework net8.0
cd ContactFunctionApp
func new --name SubmitContact --template "HTTP trigger" --authlevel "anonymous"
```

### Configure `local.settings.json`
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "UseDevelopmentStorage=true",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "SQLConnectionString": "Server=tcp:<your-server>.database.windows.net,1433;Initial Catalog=ContactDB;User ID=<user>;Password=<password>;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
  }
}
```

### Add package
```bash
dotnet add package Microsoft.Extensions.Configuration
dotnet add package Microsoft.Extensions.Configuration.Binder
dotnet add package Microsoft.Extensions.Configuration.Json
dotnet add package System.Data.SqlClient
dotnet restore
```

### Sample implementation (`Functions/SubmitContact.cs`)
```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Azure.Functions.Worker.Http;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Configuration;
using System.Net;
using System.Data.SqlClient;
using System.Text.Json;
public class ContactData
{
    public string Name { get; set; }
    public string Email { get; set; }
    public string Message { get; set; }
}
public class SubmitContact
{
    private readonly ILogger _logger;
    private readonly IConfiguration _config;
    public SubmitContact(ILoggerFactory loggerFactory, IConfiguration config)
    {
        _logger = loggerFactory.CreateLogger<SubmitContact>();
        _config = config;
    }
    [Function("SubmitContact")]
    public async Task<HttpResponseData> Run(
        [HttpTrigger(AuthorizationLevel.Anonymous, "post")] HttpRequestData req)
    {
        var contact = await req.ReadFromJsonAsync<ContactData>();
        if (contact == null || string.IsNullOrWhiteSpace(contact.Name))
        {
            var badResponse = req.CreateResponse(HttpStatusCode.BadRequest);
            await badResponse.WriteStringAsync("Invalid input.");
            return badResponse;
        }
        var connString = _config["SQLConnectionString"];
        using var conn = new SqlConnection(connString);
        await conn.OpenAsync();
        var cmd = new SqlCommand(
            "INSERT INTO ContactMessages (Name, Email, Message) VALUES (@Name, @Email, @Message)", conn);
        cmd.Parameters.AddWithValue("@Name", contact.Name);
        cmd.Parameters.AddWithValue("@Email", contact.Email);
        cmd.Parameters.AddWithValue("@Message", contact.Message);
        await cmd.ExecuteNonQueryAsync();
        var response = req.CreateResponse(HttpStatusCode.OK);
        await response.WriteStringAsync("Contact saved successfully.");
        return response;
    }
}
```

### Test locally
```bash
func start
```
PowerShell:
```powershell
Invoke-RestMethod -Uri "http://localhost:7071/api/SubmitContact" `
  -Method Post -ContentType "application/json" `
  -Body (@{ name = "Alice"; email = "alice@example.com"; message = "Hello" } | ConvertTo-Json)
```

---

## Exercise 2 — Image Upload → AI Tagging → Azure SQL (Blob Trigger)

### Scenario
When a user **uploads an image** to a blob container, a **Blob-triggered** function runs, calls **Azure AI Vision** to detect tags, and writes the results to **SQL**.

### Components
- **Blob Storage** — upload target (`images` container)
- **Azure Function (Blob Trigger)** — processing logic
- **Azure AI Vision (Computer Vision)** — tag/label detection
- **Azure SQL DB** — store file name, tags, timestamp

### SQL table
```sql
CREATE TABLE [dbo].[ImageTags](
    [Id] INT IDENTITY PRIMARY KEY,
    [FileName] NVARCHAR(200),
    [Tags] NVARCHAR(MAX),
    [UploadedAt] DATETIME DEFAULT GETDATE()
);
```

### Create storage + container (portal quick path)
- Storage account → **Create** (LRS, Standard)  
- After deploy → **Containers** → **+ Container** → Name: `images` → Access level: **Private**

### Create AI Vision resource (Computer Vision)
- Create resource → **Computer Vision** → region (e.g., East US/West Europe) → **S1**  
- Copy **Endpoint** and **Key**

### Create the Function
```bash
func init ImageTagsFunction --worker-runtime dotnet-isolated --target-framework net8.0
cd ImageTagsFunction
func new --name AnalyzeImage --template "Blob trigger"
# When prompted for path: images/{name}
```

### Configure `local.settings.json`
```json
{
  "IsEncrypted": false,
  "Values": {
    "AzureWebJobsStorage": "<storage-connection-string>",
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "VisionEndpoint": "https://<your-region>.api.cognitive.microsoft.com/",
    "VisionKey": "<your-vision-key>",
    "SQLConnectionString": "Server=tcp:<your-server>.database.windows.net,1433;Initial Catalog=ContactDB;User ID=<user>;Password=<password>;Encrypt=True;"
  }
}
```

### Add packages
```bash
dotnet add package Microsoft.Extensions.Configuration
dotnet add package Microsoft.Extensions.Configuration.Json
dotnet add package System.Data.SqlClient
dotnet restore
```

### Implementation (`Functions/AnalyzeImage.cs`)
```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;
using System.Net.Http.Headers;
using Microsoft.Extensions.Configuration;
using System.Data.SqlClient;
using System.Text.Json;

public class AnalyzeImage
{
    private readonly ILogger _logger;
    private readonly IConfiguration _config;
    private readonly HttpClient _httpClient;

    public AnalyzeImage(ILoggerFactory loggerFactory, IConfiguration config)
    {
        _logger = loggerFactory.CreateLogger<AnalyzeImage>();
        _config = config;
        _httpClient = new HttpClient();
    }

    [Function("AnalyzeImage")]
    public async Task Run(
        [BlobTrigger("images/{name}", Connection = "AzureWebJobsStorage")] Stream image,
        string name)
    {
        _logger.LogInformation($"📸 New image uploaded: {name}");

        // Prepare Vision API request
        var visionEndpoint = _config["VisionEndpoint"];
        var visionKey = _config["VisionKey"];

        var requestUrl = $"{visionEndpoint}/vision/v3.2/tag";

        var content = new StreamContent(image);
        content.Headers.ContentType = new MediaTypeHeaderValue("application/octet-stream");

        _httpClient.DefaultRequestHeaders.Clear();
        _httpClient.DefaultRequestHeaders.Add("Ocp-Apim-Subscription-Key", visionKey);

        var response = await _httpClient.PostAsync(requestUrl, content);
        response.EnsureSuccessStatusCode();

        var resultJson = await response.Content.ReadAsStringAsync();
        using var doc = JsonDocument.Parse(resultJson);
        var tags = doc.RootElement.GetProperty("tags")
                    .EnumerateArray()
                    .Select(t => t.GetProperty("name").GetString())
                    .ToList();

        var tagString = string.Join(", ", tags);

        _logger.LogInformation($"🔖 Tags: {tagString}");

        // Save to SQL DB
        var connString = _config["SQLConnectionString"];
        using var conn = new SqlConnection(connString);
        await conn.OpenAsync();

        var cmd = new SqlCommand("INSERT INTO ImageTags (FileName, Tags) VALUES (@FileName, @Tags)", conn);
        cmd.Parameters.AddWithValue("@FileName", name);
        cmd.Parameters.AddWithValue("@Tags", tagString);

        await cmd.ExecuteNonQueryAsync();
        _logger.LogInformation("✅ Tags saved to database.");
    }
}

```

### Test
- Run: `func start`  
- Upload a file to `images/` (Portal, Storage Explorer, or CLI)  
- Verify logs and query SQL: `SELECT * FROM [dbo].[ImageTags];`

---

## Exercise 3 — Nightly Image Summary + Cleanup + Email (Timer Trigger + Logic App)

### Scenario
On a schedule (e.g., nightly), a **Timer** function reads rows from SQL, deletes blobs from storage, and sends a **summary email** via a **Logic App (HTTP trigger + Send email)**.

### Logic App (Consumption)
1. Create **Logic App (Consumption)** in the same region/resource group.  
2. Add trigger **When an HTTP request is received**.  
3. Use this sample payload to generate schema:
```json
{
  "summary": "Deleted 3 images: apple.jpg, car.png",
  "runTime": "2025-07-30 02:00 AM"
}
```
4. Save to get the **POST URL** (copy it).  
5. Add **Send an email (V2)** using **Outlook.com** (for personal) or **Office 365 Outlook** (for work/school), or use Gmail/SendGrid.  
   - Subject: `Daily Image Cleanup Summary`  
   - Body example (HTML):
```html
<strong>Images Deleted:</strong><br/>
@{triggerBody()?['summary']}<br/><br/>
<i>Cleanup completed at: @{triggerBody()?['runTime']}</i>
```

### Create the Function
```bash
func init CleanupFunction --worker-runtime dotnet-isolated --target-framework net8.0
cd CleanupFunction
func new --name CleanupJob --template "Timer trigger"
# Schedule: 0 0 2 * * *   (daily at 2 AM) — see more examples below
```

### Configure `local.settings.json`
```json
{
  "IsEncrypted": false,
  "Values": {
    "FUNCTIONS_WORKER_RUNTIME": "dotnet-isolated",
    "AzureWebJobsStorage": "<storage-connection-string>",
    "SQLConnectionString": "Server=tcp:<your-server>.database.windows.net,1433;Initial Catalog=ContactDB;User ID=<user>;Password=<password>;Encrypt=True;",
    "LogicAppUrl": "<logic-app-post-url>"
  }
}
```

### Add packages
```bash
dotnet add package System.Data.SqlClient
dotnet add package Azure.Storage.Blobs
dotnet add package Microsoft.Extensions.Configuration
dotnet add package Microsoft.Extensions.Configuration.Json
dotnet add package Azure.Storage.Blobs
dotnet restore
```

### Implementation (`Functions/CleanupJob.cs`)
```csharp
using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;
using Microsoft.Extensions.Configuration;
using System.Data.SqlClient;
using Azure.Storage.Blobs;
using Azure.Storage.Blobs.Specialized;
using System.Net.Http.Json;

public class CleanupJob
{
    private readonly ILogger _logger;
    private readonly IConfiguration _config;
    private readonly HttpClient _httpClient;

    public CleanupJob(ILoggerFactory loggerFactory, IConfiguration config)
    {
        _logger = loggerFactory.CreateLogger<CleanupJob>();
        _config = config;
        _httpClient = new HttpClient();
    }

    [Function("CleanupJob")]
    public async Task Run([TimerTrigger("0 */2 * * * *")] TimerInfo myTimer)
    {
        _logger.LogInformation("⏰ Timer function started at: {time}", DateTime.Now);

        // STEP 1: Read from SQL
        string summary = "";
        var connStr = _config["SQLConnectionString"];
        using (var conn = new SqlConnection(connStr))
        {
            await conn.OpenAsync();
            var cmd = new SqlCommand("SELECT FileName, Tags FROM ImageTags", conn);
            var reader = await cmd.ExecuteReaderAsync();

            while (await reader.ReadAsync())
            {
                string file = reader["FileName"].ToString();
                string tags = reader["Tags"].ToString();
                summary += $"- {file} → {tags}\n";
            }
        }

        // STEP 2: Delete blobs
        var blobConn = _config["stgfunctionsdemo001_STORAGE"];
        var blobClient = new BlobServiceClient(blobConn);
        var container = blobClient.GetBlobContainerClient("images");

        int deleteCount = 0;
        await foreach (var blob in container.GetBlobsAsync())
        {
            var blobRef = container.GetBlobClient(blob.Name);
            await blobRef.DeleteIfExistsAsync();
            deleteCount++;
        }

        // STEP 3: Send to Logic App
        var payload = new
        {
            summary = summary == "" ? "No images found." : summary,
            runTime = DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ss") + " UTC"
        };

        var logicUrl = _config["LogicAppUrl"];
        var result = await _httpClient.PostAsJsonAsync(logicUrl, payload);

        _logger.LogInformation("📤 Email summary sent. Blobs deleted: {count}", deleteCount);
    }
}
```

### Test locally
```bash
func start
```
- Should log activity, delete blobs, and cause the Logic App to send an email.

### Cron examples
- Every minute: `0 */1 * * * *`  
- Every 2 minutes: `0 */2 * * * *`  
- Every 5 minutes: `0 */5 * * * *`  
- Daily at 2:00 AM: `0 0 2 * * *`

---

## Deploying to Azure
**VS Code:**
1. Right‑click the project folder that contains `host.json` → **Deploy to Function App**  
2. Create/select a Function App (Consumption, Windows, .NET 8)  
3. In the Function App → **Configuration**, add keys used by your app:  
   - `SQLConnectionString`, `AzureWebJobsStorage`, `VisionEndpoint`/`VisionKey` (Ex2), `LogicAppUrl` (Ex3)  
4. **Restart** the Function App

**Core Tools:**
```bash
func azure functionapp publish <YourFunctionAppName>
```

---

## Troubleshooting
- **401 sending email from Logic App:** Use the correct connector (Outlook.com for personal; Office 365 Outlook for work/school). Re‑authenticate the connection.  
- **Blob trigger not firing:** Confirm the container path (e.g., `images/{name}`) and that the storage connection string is valid.  
- **SQL connectivity:** Check firewall/Private Endpoint and connection string.  
- **Missing `host.json` on deploy:** Ensure you deploy the project root (where `host.json` lives).  
- **Cold starts:** Expected on Consumption; consider Premium for lower latency.

---

## Cleanup
- Remove resource groups created for each exercise to avoid charges, or delete individual resources (SQL DBs, Storage, Logic App) as needed.
