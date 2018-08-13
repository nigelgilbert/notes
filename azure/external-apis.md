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
| **cons**    | high resources to implement, high cost to maintain |

#### Azure Function + Blob Trigger
|             |      |
|:------------|:-----|
| **layer**   | ingestion phase of ETL pipeline |
| **pros**    | binary blobs = *anything*– images, records; occurs before Azure SQL, hence well-suited for e.g. formatting / processing; Azure makes blobs are very easy to integrate with |
| **cons**    | high resources to implement, can be a chore to map entities out of binary |

#### Azure Function + Azure SQL Polling
|             |      |
|:------------|:-----|
| **layer**   | warehouse phase of ETL pipeline |
| **pros**    | blobs = *anything*– images, records; occurs before Azure SQL, hence well-suited for e.g. formatting / processing; Azure makes blobs are very easy integration |
| **cons**    | only possible in Azure SQL (max size 4TB), not Azure Data Warehouse |

#### Azure Data Factory v2 + Web Activities
|             |      |
|:------------|:-----|
| **layer**   | ingestion phase of ETL pipeline |
| **pros**    | low development costs |
| **cons**    | very new so few docs; some features aren't supported |

https://predica.pl/blog/adf-v2-conditional-execution-parameters/
https://docs.microsoft.com/en-us/azure/data-factory/control-flow-web-activity

#### Azure Data Factory v1 + Custom Activities
|             |      |
|:------------|:-----|
| **layer**   | ingestion phase of ETL pipeline |
| **pros**    | medium development costs |
| **cons**    | requires software development |

https://docs.microsoft.com/en-us/azure/data-factory/transform-data-using-dotnet-custom-activity

