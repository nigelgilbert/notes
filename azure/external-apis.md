# propogating database record changes to 3rd party API's from Azure pipelines and databases

## Scenario

A client has a database of merchandise items.  They keep records in their own database, but also need to update a 3rd party e-commerce site when inventory changes using a restful API. Items maybe added or removed, and the client wants those changes automatically reflected in their e-commerce profile.

**problem:** How do you keep the 3rd party's records in sync with your database?

This article explores some of the options available for propogating database record changes to an external API using Azure Cloud Services.

## Architecture Context

The client has a [data pipeline](https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/data/images/enterprise-bi-sqldw-adf.png) that pulls records from their ERP software, as well as a [number of azure functions](http://redblacksoftware.co.uk/wp-content/uploads/2017/09/RedBlack-and-Azure.png) which automate manual processes and feed Power BI Reports.

## Strategies

#### SQL Server Assemblies
|             |      |
|:------------|:-----|
| **layer**   | on-prem |
| **pros**    | integrate w/ old stacks |
| **cons**    | not supported in azure sql, business logic in the db |

[This technique](https://blogs.msdn.microsoft.com/amitagarwal/2018/01/11/azure-function-apps-trigger-in-azure-sql-sql-server-to-execute-azure-function/) uses CLR assemblies in SQL Server to invoke an Azure function's HTTP Trigger.  The Azure function then calls the 3rd party's API to update records, while handling error logging.  CLR assemblies are [not available for Azure SQL](https://stackoverflow.com/a/37342653), so this is only applicable to on-premise or Window Server VM's.

#### Application Business Logic
|             |      |
|:------------|:-----|
| **layer**   | on-prem or Azure-deployed apps |
| **pros**    | custom-fit solutions |
| **cons**    | high resources to implement, high cost to maintain, highly coupled to app or EPR software |

For ERP, the software to update records using the 3rd party API would be implemented in X++.  This also may be .NET or any other application running on Azure App Service.

#### Azure Function + Blob Trigger
|             |      |
|:------------|:-----|
| **layer**   | ingestion phase of ETL pipeline |
| **pros**    | binary blobs = *anything*– images, records; occurs before Azure SQL, hence well-suited for e.g. formatting / processing; Azure makes blobs are very easy to integrate with |
| **cons**    | high resources to implement, can be a chore to map entities out of binary |

Azure Functions have excellent blob storage support, which means using blob storage to triggering Functions is trivial.  This is an ideal place for e.g. sync'ing images or other files w/ a 3rd party API.  The issue is that one has to be intentional with data formats when staging to binary so that they can be easily read by the Functions.  Moreover, you waste resource turning the binary data into entities.

#### Azure Function + Azure SQL Polling
|             |      |
|:------------|:-----|
| **layer**   | warehouse phase of ETL pipeline |
| **pros**    | blobs = *anything*– images, records; occurs before Azure SQL, hence well-suited for e.g. formatting / processing; Azure makes blobs are very easy integration |
| **cons**    | only possible in Azure SQL (max [size 4TB](https://www.jamesserra.com/archive/2017/03/sql-db-now-supports-4tb-max-database-size/)), not Azure Data Warehouse |

This technique is nice because one doesn't have to worry about data formatting– it's easier to read records out of normalized tables.  Also, it's sometimes more reliable to operate on data that is already in the SQL database (e.g. if binary is only a staging area and some records are filtered out in the pipeline's transform processes). The primary issue with this technique is that it's only viable in Azure SQL, which has a limited max-size.

#### Azure Data Factory v1 + Custom Activities
|             |      |
|:------------|:-----|
| **layer**   | ingestion phase of ETL pipeline |
| **pros**    | medium development costs |
| **cons**    | requires custom software solution |

Azure Data Factories lets one move and transform data between [supported](https://docs.microsoft.com/en-us/azure/data-factory/copy-activity-overview#supported-data-stores-and-formats) source and sink data sources.  To move data to/from an unsupported store, one can use [custom activities](https://docs.microsoft.com/en-us/azure/data-factory/transform-data-using-dotnet-custom-activity).

#### Azure Data Factory v2 + Web Activities
|             |      |
|:------------|:-----|
| **layer**   | ingestion phase of ETL pipeline |
| **pros**    | low development costs, no coding required |
| **cons**    | very new so few docs; some features aren't supported |

[Azure Data Factory v2](https://docs.microsoft.com/en-us/azure/data-factory/compare-versions) has the same function as ADFv1, however it also provides powerful non-coding [control flow](https://predica.pl/blog/adf-v2-conditional-execution-parameters/), which uses a block-based GUI to [visually build pipelines.](https://azure.microsoft.com/en-us/resources/videos/azure-friday-visually-build-pipelines-for-azure-data-factory-v2/)  You can use [ADFv2 Web Activities](https://docs.microsoft.com/en-us/azure/data-factory/control-flow-web-activity) to upload data to the 3rd party API.