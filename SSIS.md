# SSIS Implementation Guide for SFTP File Processing Solution

## Overview
This guide provides step-by-step instructions for implementing SSIS packages that work with the Azure SFTP file processing solution.

## Prerequisites

### Software Requirements
- SQL Server Data Tools (SSDT) or Visual Studio with SSIS extension
- SQL Server Integration Services (SSIS) runtime
- Azure Storage Explorer (for testing)
- .NET Framework 4.7.2 or higher

### Azure Components (Deployed via ARM Template)
- Azure Storage Account with SFTP enabled
- Azure SQL Database with SSISDB and ProcessingLogs databases
- Azure Key Vault for encryption keys
- Azure Logic Apps for file monitoring
- Azure Monitor for logging and alerting

## SSIS Package Architecture

### Package Structure
```
SFTPFileProcessing.dtsx (Master Package)
├── Data Flow Task: Process File
├── Script Task: Log Processing Start
├── Script Task: Move File to Processing
├── Data Flow Task: Transform Data
├── Script Task: Encrypt/Decrypt File
├── Script Task: Move to Outbound/Archive
├── Script Task: Log Processing Complete
└── Script Task: Error Handling
```

## Step 1: Create SSIS Project

1. Open Visual Studio/SSDT
2. Create new Integration Services Project
3. Name it "SFTPFileProcessingSolution"

## Step 2: Configure Project Parameters

Create the following project parameters:

| Parameter Name | Data Type | Value | Description |
|----------------|-----------|-------|-------------|
| StorageConnectionString | String | Connection string | Azure Storage connection |
| LoggingConnectionString | String | Connection string | SQL Database connection |
| KeyVaultUrl | String | URL | Azure Key Vault URL |
| InboundContainer | String | inbound | Source container name |
| OutboundContainer | String | outbound | Destination container name |
| ProcessingContainer | String | processing | Processing container name |
| ArchiveContainer | String | archive | Archive container name |
| ErrorContainer | String | error | Error container name |

## Step 3: Create Connection Managers

### 1. Azure Storage Connection Manager
- Type: AzureStorage
- Name: AzureStorageConnectionManager
- Connection String: Use StorageConnectionString parameter

### 2. SQL Server Connection Manager
- Type: OLEDB
- Name: LoggingDatabaseConnectionManager
- Connection String: Use LoggingConnectionString parameter

### 3. Azure Key Vault Connection Manager (Custom)
- Type: HTTP Connection Manager
- Name: KeyVaultConnectionManager
- Server URL: Use KeyVaultUrl parameter

## Step 4: Main Processing Package (SFTPFileProcessing.dtsx)

### Package Variables
```xml
<!-- Create these package variables -->
<Variable Name="FileName" DataType="String" />
<Variable Name="FileSize" DataType="Int64" />
<Variable Name="LogId" DataType="Int64" />
<Variable Name="ProcessingStatus" DataType="String" />
<Variable Name="ErrorMessage" DataType="String" />
<Variable Name="SourceBlobPath" DataType="String" />
<Variable Name="DestinationBlobPath" DataType="String" />
```

### Control Flow Tasks

#### 1. Script Task: Initialize Variables
```csharp
public void Main()
{
    try
    {
        // Get file information from package parameters
        string fileName = Dts.Variables["$Package::FileName"].Value.ToString();
        Dts.Variables["User::FileName"].Value = fileName;
        
        // Construct blob paths
        string inboundContainer = Dts.Variables["$Project::InboundContainer"].Value.ToString();
        Dts.Variables["User::SourceBlobPath"].Value = $"{inboundContainer}/{fileName}";
        
        Dts.TaskResult = (int)ScriptResults.Success;
    }
    catch (Exception ex)
    {
        Dts.Variables["User::ErrorMessage"].Value = ex.Message;
        Dts.TaskResult = (int)ScriptResults.Failure;
    }
}
```

#### 2. Execute SQL Task: Log Processing Start
```sql
-- SQL Command
EXEC [dbo].[sp_LogFileProcessingStart] 
    @FileName = ?, 
    @FileSize = ?, 
    @SourceContainer = ?, 
    @SSISPackageName = ?, 
    @LogId = ? OUTPUT

-- Parameter Mapping
Parameter 0: Input, VARCHAR, User::FileName
Parameter 1: Input, BIGINT, User::FileSize
Parameter 2: Input, VARCHAR, $Project::InboundContainer
Parameter 3: Input, VARCHAR, System::PackageName
Parameter 4: Output, BIGINT, User::LogId
```

#### 3. Script Task: Move File to Processing Container
```csharp
#region Namespaces
using System;
using Microsoft.WindowsAzure.Storage;
using Microsoft.WindowsAzure.Storage.Blob;
#endregion

public void Main()
{
    try
    {
        string connectionString = Dts.Variables["$Project::StorageConnectionString"].Value.ToString();
        string fileName = Dts.Variables["User::FileName"].Value.ToString();
        string inboundContainer = Dts.Variables["$Project::InboundContainer"].Value.ToString();
        string processingContainer = Dts.Variables["$Project::ProcessingContainer"].Value.ToString();
        
        CloudStorageAccount storageAccount = CloudStorageAccount.Parse(connectionString);
        CloudBlobClient blobClient = storageAccount.CreateCloudBlobClient();
        
        // Get source blob
        CloudBlobContainer sourceContainer = blobClient.GetContainerReference(inboundContainer);
        CloudBlockBlob sourceBlob = sourceContainer.GetBlockBlobReference(fileName);
        
        // Get destination blob
        CloudBlobContainer destContainer = blobClient.GetContainerReference(processingContainer);
        CloudBlockBlob destBlob = destContainer.GetBlockBlobReference(fileName);
        
        // Copy blob
        destBlob.StartCopy(sourceBlob);
        
        // Wait for copy to complete
        while (destBlob.CopyState.Status == CopyStatus.Pending)
        {
            System.Threading.Thread.Sleep(1000);
            destBlob.FetchAttributes();
        }
        
        if (destBlob.CopyState.Status == CopyStatus.Success)
        {
            // Delete source blob
            sourceBlob.DeleteIfExists();
            
            // Update variables
            Dts.Variables["User::SourceBlobPath"].Value = $"{processingContainer}/{fileName}";
            Dts.Variables["User::FileSize"].Value = destBlob.Properties.Length;
        }
        
        Dts.TaskResult = (int)ScriptResults.Success;
    }
    catch (Exception ex)
    {
        Dts.Variables["User::ErrorMessage"].Value = ex.Message;
        Dts.TaskResult = (int)ScriptResults.Failure;
    }
}
```

#### 4. Data Flow Task: Process File Content

**Data Flow Components:**
1. **Azure Blob Source**: Read file from processing container
2. **Data Conversion**: Convert data types as needed
3. **Conditional Split**: Route different file types
4. **Script Component**: Custom data transformation
5. **Azure Blob Destination**: Write processed file

**Sample Script Component (Transformation):**
```csharp
public override void Input0_ProcessInputRow(Input0Buffer Row)
{
    try
    {
        // Example: Encrypt sensitive data
        if (!Row.SensitiveData_IsNull && !string.IsNullOrEmpty(Row.SensitiveData))
        {
            Row.SensitiveData = EncryptData(Row.SensitiveData);
        }
        
        // Add processing timestamp
        Row.ProcessedDateTime = DateTime.UtcNow;
        
        // Set processing status
        Row.ProcessingStatus = "PROCESSED";
    }
    catch (Exception ex)
    {
        Row.ProcessingStatus = "ERROR";
        ComponentMetaData.FireError(0, "ProcessFile", ex.Message, "", 0, out bool cancel);
    }
}

private string EncryptData(string data)
{
    // Implement encryption logic using Azure Key Vault
    // This is a placeholder - implement actual encryption
    return Convert.ToBase64String(System.Text.Encoding.UTF8.GetBytes(data));
}
```

#### 5. Script Task: Move to Final Destination
```csharp
public void Main()
{
    try
    {
        string connectionString = Dts.Variables["$Project::StorageConnectionString"].Value.ToString();
        string fileName = Dts.Variables["User::FileName"].Value.ToString();
        string processingContainer = Dts.Variables["$Project::ProcessingContainer"].Value.ToString();
        string outboundContainer = Dts.Variables["$Project::OutboundContainer"].Value.ToString();
        string archiveContainer = Dts.Variables["$Project::ArchiveContainer"].Value.ToString();
        
        CloudStorageAccount storageAccount = CloudStorageAccount.Parse(connectionString);
        CloudBlobClient blobClient = storageAccount.CreateCloudBlobClient();
        
        // Get processed file
        CloudBlobContainer sourceContainer = blobClient.GetContainerReference(processingContainer);
        CloudBlockBlob sourceBlob = sourceContainer.GetBlockBlobReference(fileName);
        
        // Copy to outbound
        CloudBlobContainer outboundCont = blobClient.GetContainerReference(outboundContainer);
        CloudBlockBlob outboundBlob = outboundCont.GetBlockBlobReference(fileName);
        outboundBlob.StartCopy(sourceBlob);
        
        // Copy to archive with timestamp
        string archiveFileName = $"{DateTime.UtcNow:yyyyMMdd_HHmmss}_{fileName}";
        CloudBlobContainer archiveCont = blobClient.GetContainerReference(archiveContainer);
        CloudBlockBlob archiveBlob = archiveCont.GetBlockBlobReference(archiveFileName);
        archiveBlob.StartCopy(sourceBlob);
        
        // Wait for copies to complete
        WaitForCopyCompletion(outboundBlob);
        WaitForCopyCompletion(archiveBlob);
        
        // Delete from processing
        sourceBlob.DeleteIfExists();
        
        Dts.Variables["User::ProcessingStatus"].Value = "COMPLETED";
        Dts.Variables["User::DestinationBlobPath"].Value = $"{outboundContainer}/{fileName}";
        
        Dts.TaskResult = (int)ScriptResults.Success;
    }
    catch (Exception ex)
    {
        Dts.Variables["User::ErrorMessage"].Value = ex.Message;
        Dts.Variables["User::ProcessingStatus"].Value = "FAILED";
        Dts.TaskResult = (int)ScriptResults.Failure;
    }
}

private void WaitForCopyCompletion(CloudBlockBlob blob)
{
    while (blob.CopyState.Status == CopyStatus.Pending)
    {
        System.Threading.Thread.Sleep(1000);
        blob.FetchAttributes();
    }
}
```

#### 6. Execute SQL Task: Log Processing Complete
```sql
-- SQL Command
EXEC [dbo].[sp_LogFileProcessingComplete] 
    @LogId = ?, 
    @Status = ?, 
    @DestinationContainer = ?, 
    @ErrorMessage = ?

-- Parameter Mapping
Parameter 0: Input, BIGINT, User::LogId
Parameter 1: Input, VARCHAR, User::ProcessingStatus
Parameter 2: Input, VARCHAR, $Project::OutboundContainer
Parameter 3: Input, VARCHAR, User::ErrorMessage
```

## Step 5: Error Handling Package

Create a separate package for error handling:

### ErrorHandling.dtsx
```csharp
// Script Task: Move Failed File to Error Container
public void Main()
{
    try
    {
        string connectionString = Dts.Variables["$Project::StorageConnectionString"].Value.ToString();
        string fileName = Dts.Variables["User::FileName"].Value.ToString();
        string errorContainer = Dts.Variables["$Project::ErrorContainer"].Value.ToString();
        
        // Move file to error container
        // Add error details to blob metadata
        // Send error notification
        
        Dts.TaskResult = (int)ScriptResults.Success;
    }
    catch (Exception ex)
    {
        Dts.TaskResult = (int)ScriptResults.Failure;
    }
}
```

## Step 6: Package Configuration

### Environment Configuration
Create SSIS environments for different deployment stages:

1. **Development Environment**
   - Local storage emulator
   - Local SQL Server

2. **Production Environment**
   - Azure Storage Account
   - Azure SQL Database

### Security Configuration
1. Store sensitive parameters in SSIS catalog
2. Use Azure Key Vault for encryption keys
3. Configure service accounts with minimal permissions

## Step 7: Deployment

### 1. Deploy to SSIS Catalog
```sql
-- Create SSIS Catalog if not exists
EXEC catalog.create_catalog @password = N'YourCatalogPassword'

-- Create folder
EXEC catalog.create_folder @folder_name = N'SFTPFileProcessing'

-- Deploy project (use SSDT deployment wizard)
```

### 2. Configure Package Execution
```sql
-- Create environment
EXEC catalog.create_environment 
    @environment_name = N'Production',
    @folder_name = N'SFTPFileProcessing'

-- Add environment variables
EXEC catalog.create_environment_variable
    @environment_name = N'Production',
    @folder_name = N'SFTPFileProcessing',
    @variable_name = N'StorageConnectionString',
    @data_type = N'String',
    @sensitive = 1,
    @value = N'YourStorageConnectionString'
```

## Step 8: Integration with Logic Apps

### REST API Endpoint for Package Execution
Create a web service or Azure Function to trigger SSIS packages:

```csharp
[HttpPost]
public async Task<IActionResult> TriggerSSISPackage([FromBody] FileProcessingRequest request)
{
    try
    {
        var connectionString = "Server=your-sql-server;Database=SSISDB;Integrated Security=true";
        
        using (var connection = new SqlConnection(connectionString))
        {
            var command = new SqlCommand("catalog.start_execution", connection);
            command.CommandType = CommandType.StoredProcedure;
            
            command.Parameters.AddWithValue("@execution_id", SqlDbType.BigInt).Direction = ParameterDirection.Output;
            command.Parameters.AddWithValue("@folder_name", "SFTPFileProcessing");
            command.Parameters.AddWithValue("@project_name", "SFTPFileProcessingSolution");
            command.Parameters.AddWithValue("@package_name", "SFTPFileProcessing.dtsx");
            
            // Add package parameters
            command.Parameters.AddWithValue("@parameter_set_id", SqlDbType.BigInt).Value = DBNull.Value;
            
            await connection.OpenAsync();
            await command.ExecuteNonQueryAsync();
            
            var executionId = (long)command.Parameters["@execution_id"].Value;
            
            return Ok(new { ExecutionId = executionId, Status = "Started" });
        }
    }
    catch (Exception ex)
    {
        return BadRequest(new { Error = ex.Message });
    }
}
```

## Step 9: Monitoring and Maintenance

### Performance Monitoring
1. Monitor SSIS execution reports
2. Set up Azure Monitor alerts
3. Track file processing metrics

### Maintenance Tasks
1. Regular cleanup of old log entries
2. Archive processed files
3. Monitor storage capacity
4. Update encryption keys periodically

## Troubleshooting Common Issues

### Connection Issues
- Verify firewall rules for Azure SQL Database
- Check storage account access keys
- Validate Key Vault permissions

### Performance Issues
- Optimize data flow buffer sizes
- Use parallel processing for large files
- Implement file size limits

### Security Issues
- Rotate access keys regularly
- Monitor failed authentication attempts
- Implement least privilege access

## Testing Checklist

- [ ] File upload via SFTP works
- [ ] Logic App triggers correctly
- [ ] SSIS package executes successfully
- [ ] Files are processed and moved correctly
- [ ] Error handling works as expected
- [ ] Logging captures all events
- [ ] Email notifications are sent
- [ ] Monitoring alerts are triggered
- [ ] Archive process works correctly
- [ ] Security permissions are correct
