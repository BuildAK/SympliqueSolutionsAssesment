# SympliqueSolutionsAssesment

Azure Serverless Cost Optimization Solution for Cosmos DB Billing Records
Solution Overview
I propose implementing a tiered storage strategy with Azure Cosmos DB's built-in partitioning capabilities combined with Azure Blob Storage for cold data. This approach maintains API contracts while significantly reducing costs for rarely accessed data.

Key Components:
Partitioning Strategy: Use time-based partitioning in Cosmos DB

Hot/Cold Data Separation: Active data in Cosmos DB, archived data in Blob Storage

Serverless Retrieval: Azure Functions to handle seamless data retrieval from both sources

Automated Archiving: Logic App or Function to move old records

Architecture Diagram

[Client Application] 
       ↓
[API Layer (Existing)] → [Azure Function (Proxy)]
       |                          |
[Cosmos DB (Hot Data)]    [Blob Storage (Cold Data)]
       ↑                          ↑
[Archiving Function] ← [Time-Based Trigger]
Implementation Details
1. Partition Key Strategy

// Example record structure with effective partition key
public class BillingRecord
{
    [JsonProperty("id")]
    public string Id { get; set; }
    
    public DateTime BillingDate { get; set; }
    
    // Partition key combining year and month
    public string PartitionKey => $"billing_{BillingDate:yyyy_MM}";
    
    // Other billing record properties...
}
2. Automated Archiving Process

# PowerShell script to set up Cosmos DB change feed for archiving
az cosmosdb sql container create `
    --resource-group $resourceGroup `
    --account-name $cosmosAccount `
    --database-name $databaseName `
    --name $containerName `
    --partition-key-path "/PartitionKey" `
    --throughput 400
3. Archiving Azure Function

[FunctionName("ArchiveOldRecords")]
public static async Task Run(
    [TimerTrigger("0 0 1 * * *")] TimerInfo timer, // Runs at 1am daily
    [CosmosDB(/* hot data connection */)] DocumentClient client,
    [BlobStorage(/* cold data connection */)] CloudBlobContainer blobContainer,
    ILogger log)
{
    var threeMonthsAgo = DateTime.UtcNow.AddMonths(-3);
    var query = client.CreateDocumentQuery<BillingRecord>(
        UriFactory.CreateDocumentCollectionUri("billing", "records"),
        new FeedOptions { EnableCrossPartitionQuery = true })
        .Where(r => r.BillingDate < threeMonthsAgo)
        .AsDocumentQuery();

    while (query.HasMoreResults)
    {
        var batch = await query.ExecuteNextAsync<BillingRecord>();
        foreach (var record in batch)
        {
            // Upload to blob storage
            var blobName = $"{record.PartitionKey}/{record.Id}.json";
            var blob = blobContainer.GetBlockBlobReference(blobName);
            await blob.UploadTextAsync(JsonConvert.SerializeObject(record));
            
            // Delete from Cosmos DB
            await client.DeleteDocumentAsync(
                UriFactory.CreateDocumentUri("billing", "records", record.Id),
                new RequestOptions { PartitionKey = new PartitionKey(record.PartitionKey) });
        }
    }
}
4. Unified Retrieval Function

[FunctionName("GetBillingRecord")]
public static async Task<IActionResult> Run(
    [HttpTrigger(AuthorizationLevel.Function, "get", Route = "records/{id}")] HttpRequest req,
    string id,
    [CosmosDB(/* hot data */)] DocumentClient client,
    [BlobStorage(/* cold data */)] CloudBlobContainer blobContainer,
    ILogger log)
{
    try
    {
        // First try Cosmos DB (hot data)
        var response = await client.ReadDocumentAsync(
            UriFactory.CreateDocumentUri("billing", "records", id),
            new RequestOptions { PartitionKey = new PartitionKey($"billing_{DateTime.UtcNow:yyyy_MM}") });
        
        return new OkObjectResult(response.Resource);
    }
    catch (DocumentClientException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
    {
        // If not found, check blob storage (cold data)
        var blobs = await blobContainer.ListBlobsSegmentedAsync(
            prefix: $"billing_",
            useFlatBlobListing: true,
            blobListingDetails: BlobListingDetails.None,
            maxResults: 1,
            cancellationToken: default(CancellationToken));
        
        if (blobs.Results.Count == 0)
            return new NotFoundResult();
            
        var blob = (CloudBlockBlob)blobs.Results.First();
        var content = await blob.DownloadTextAsync();
        return new OkObjectResult(content);
    }
}
Cost Optimization Analysis
Before Optimization
Cosmos DB: 2M records × 300KB ≈ 600GB

Estimated cost: $600/month (at $1/GB)

After Optimization
Cosmos DB: Only recent 3 months (~25% of data) ≈ 150GB → $150/month

Blob Storage: 450GB in Cool Tier → ~$9/month

Azure Functions: Minimal cost (~$0.50/month)

Total Savings: ~$440/month (73% reduction)

#Implementation Steps
Prepare Blob Storage


az storage account create --name billingarchive --resource-group myRG --sku Standard_LRS --access-tier cool
az storage container create --name billing-records --account-name billingarchive
Deploy Archiving Function


func azure functionapp publish billing-archive-function
Update Retrieval Logic


func azure functionapp publish billing-api-function
Monitor Transition


az cosmosdb show --name billingdb --resource-group myRG --query "documentEndpoint"
az monitor metrics list --resource billingarchive --metric "BlobCapacity"
Maintenance Considerations
Monitoring: Set up alerts for archive retrieval latency

Lifecycle Management: Configure blob storage lifecycle to move to archive tier after 1 year

Testing: Validate retrieval performance meets SLAs

Documentation: Update runbooks for support teams

This solution provides a balance between cost optimization and maintainability while meeting all your requirements for no API changes, no downtime, and no data loss.
