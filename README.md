# Tutorial: Migrate SQL Server to Azure SQL Managed Instance using Log Replay Service (Preview)

This template allows you to create a SQL Server instance on Virtual Machine which acts as a source for migration and a Azure SQL Managed instance inside a new virtual network. Storage account is already provisioned which is required to copy and transfer SQL Server backups between source and target.


# Solution overview and deployed resources. 

In this tutorial, you will deploy resources required for the migration. Then, you migrate the AdventureWorks database from a source self-hosted SQL Server instance to a target Azure SQL Server Managed Instance with minimal downtime by using log replay service.


## Target audience

- Infrastructure Architect
- Application Developer
- IT Professional
- Cloud Solution Architect

# Architecture

The [Template.json](https://github.com/Ganapathivarma07/LRS-Migration-AzureSQLMI/blob/master/template.json) Azure Resource Manager template will help you automatically deploy the diagram below, which includes:

- SQL Server instance on Azure VM.
- Azure SQL managed instance inside a virtual network
- A storage account


![alt image](https://github.com/Ganapathivarma07/LRS-Migration-AzureSQLMI/blob/c65c38f40aa626403e011e6bf2a28e647ee5a102/Images/log-replay-service-Architecture.png)



[Template.json](https://github.com/Ganapathivarma07/LRS-Migration-AzureSQLMI/blob/master/template.json) can be modified to match your current infrastructure needs.

## One Click Deploying Template
<!-- Powershell command for Translating Git URL for template.json
    $url = "https://raw.githubusercontent.com/Ganapathivarma07/LRS-Migration-AzureSQLMI/master/template.json"
    [uri]::EscapeDataString($url)
    >> uri = https%3A%2F%2Fgithub.com%2FGanapathivarma07%2FLRS-Migration-AzureSQLMI%2Fblob%2F
master%2Ftemplate.json

Base URL: https://portal.azure.com/#create/Microsoft.Template/uri
Final URL: <Base URL>/<uri>
-->
[![Deploy to Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2FGanapathivarma07%2FLRS-Migration-AzureSQLMI%2Fmaster%2Ftemplate.json)


## Deploying an ARM Template using the Azure portal

- Visit https://portal.azure.com

- Allow 30 minutes for the deployment to complete

## Azure services and related products

- Azure Blob storage
- Azure Virutal machine
- Azure SQL Managed Instance


## Pre-requisites:

Consider the requirements in this section to get started with using LRS to migrate. 

### SQL Server 

Make sure you have the following requirements for SQL Server: 

- SQL Server versions 2008 to 2019
- Full backup of databases (one or multiple files)
- Differential backup (one or multiple files)
- Log backup (not split for a transaction log file)
- `CHECKSUM` enabled for backups (mandatory)

### Azure 

Make sure you have the following requirements for Azure: 

- PowerShell Az.SQL module version 2.16.0 or later ([installed](https://www.powershellgallery.com/packages/Az.Sql/) or accessed through [Azure Cloud Shell](/azure/cloud-shell/))
- Azure CLI version 2.19.0 or later ([installed](/cli/azure/install-azure-cli))
- Azure Blob Storage container provisioned
- Shared access signature (SAS) security token with read and list permissions generated for the Blob Storage container

### Azure RBAC permissions

Running LRS through the provided clients requires one of the following Azure roles:
- Subscription Owner role
- [SQL Managed Instance Contributor](../../role-based-access-control/built-in-roles.md#sql-managed-instance-contributor) role
- Custom role with the following permission: `Microsoft.Sql/managedInstances/databases/*`

## Requirements

Please ensure the following requirements are met:
- Use the full recovery model on SQL Server (mandatory).
- Use `CHECKSUM` for backups on SQL Server (mandatory).
- Place backup files for an individual database inside a separate folder in a flat-file structure (mandatory). Nested folders inside database folders are not supported.
- Plan to complete the migration within 36 hours after you start LRS (mandatory). This is a grace period during which system-managed software patches are postponed.

## Deployment steps

1. Create Azure blob container in a storage account

Follow these steps to Create Azure blob container 
a. To create a container, expand the storage account you created in the proceeding step. 
b. Select Blob Containers, right-click and select Create Blob Container. 
c. Enter the name for your blob container.

2. Migration steps

Follow these steps in the diagram below to start the migration using log replay service

:::image type="content" source="./media/log-replay-service-migrate/log-replay-service-conceptual.png" alt-text="Diagram that explains the Log Replay Service orchestration steps for SQL Managed Instance." border="false":::
	
| Operation | Details |
| :----------------------------- | :------------------------- |
| **1. Copy database backups from SQL Server to Blob Storage**. | Copy full, differential, and log backups from SQL Server to a Blob Storage container by using [AzCopy](../../storage/common/storage-use-azcopy-v10.md) or [Azure Storage Explorer](https://azure.microsoft.com/features/storage-explorer/). <br /><br />Use any file names. LRS doesn't require a specific file-naming convention.<br /><br />Use a separate folder for each database when migrating several databases. |
| **2. Start LRS in the cloud**. | You can start the service with PowerShell ([start-azsqlinstancedatabaselogreplay](/powershell/module/az.sql/start-azsqlinstancedatabaselogreplay)) or the Azure CLI ([az_sql_midb_log_replay_start cmdlets](/cli/azure/sql/midb/log-replay#az-sql-midb-log-replay-start)). <br /><br /> Start LRS separately for each database that points to a backup folder on Blob Storage. <br /><br /> After the service starts, it will take backups from the Blob Storage container and start restoring them to SQL Managed Instance.<br /><br /> When started in continuous mode, LRS restores all the  backups initially uploaded and then watches for any new files uploaded to the folder. The service will continuously apply logs based on the log sequence number (LSN) chain until it's stopped manually. |
| **2.1. Monitor the operation's progress**. | You can monitor progress of the restore operation with PowerShell ([get-azsqlinstancedatabaselogreplay](/powershell/module/az.sql/get-azsqlinstancedatabaselogreplay)) or the Azure CLI ([az_sql_midb_log_replay_show cmdlets](/cli/azure/sql/midb/log-replay#az-sql-midb-log-replay-show)). |
| **2.2. Stop the operation if needed**. | If you need to stop the migration process, use PowerShell ([stop-azsqlinstancedatabaselogreplay](/powershell/module/az.sql/stop-azsqlinstancedatabaselogreplay)) or the Azure CLI ([az_sql_midb_log_replay_stop](/cli/azure/sql/midb/log-replay#az-sql-midb-log-replay-stop)). <br /><br /> Stopping the operation deletes the database that you're restoring to SQL Managed Instance. After you stop an operation, you can't resume LRS for a database. You need to restart the migration process from the beginning. |
| **3. Cut over to the cloud when you're ready**. | Stop the application and workload. Take the last log-tail backup and upload it to Azure Blob Storage.<br /><br /> Complete the cutover by initiating an LRS `complete` operation with PowerShell ([complete-azsqlinstancedatabaselogreplay](/powershell/module/az.sql/complete-azsqlinstancedatabaselogreplay)) or the Azure CLI [az_sql_midb_log_replay_complete](/cli/azure/sql/midb/log-replay#az-sql-midb-log-replay-complete). This operation stops LRS and brings the database online for read and write workloads on SQL Managed Instance.<br /><br /> Repoint the application connection string from SQL Server to SQL Managed Instance. You will need to orchestrate this step yourself, either through a manual connection string change in your application, or automatically (for example, if your application can read the connection string from a property, or a database). |

## Best practices

We recommend the following best practices:
- Run [Data Migration Assistant](/sql/dma/dma-overview) to validate that your databases are ready to be migrated to SQL Managed Instance. 
- Split full and differential backups into multiple files, instead of using a single file.
- Enable backup compression to help the network transfer speeds.
- Use Cloud Shell to run PowerShell or CLI scripts, because it will always be updated to the latest cmdlets released.

## Related references

1.	https://docs.microsoft.com/en-gb/azure/azure-sql/managed-instance/log-replay-service-migrate
2.	https://docs.microsoft.com/en-us/sql/relational-databases/backup-restore/backup-overview-sql-server?view=sql-server-ver15
3.	https://docs.microsoft.com/en-us/azure/storage/blobs/quickstart-storage-explorer
4.	https://docs.microsoft.com/en-us/azure/storage/common/storage-use-azcopy-v10
5.	https://techcommunity.microsoft.com/t5/azure-sql-blog/migrate-databases-from-sql-server-to-sql-managed-instance-using/ba-p/2144303


## License & Contribute

You are responsible for the performance, the necessary testing, and if needed any regulatory clearance for any of the models produced by this toolbox.
Please refer [LICENSE](LICENSE) &  [Contribute](https://github.com/Ganapathivarma07/LRS-Migration-AzureSQLMI/blob/master/Contribute.md) for more details


