# Azure Unused Resource Finder

The Azure Unused Resource Finder shortly known as "Scavengers" is designed to help Azure cloud consumers to find the cost saving opportunities of their Infrastructure. It can perform assessment of multiple subscriptions at a time on different tenants of your Azure resources. It works based on Microsoft's cost optimization best practices defined in the Well-Architected and Cloud Adoption Framework.

Assess your Azure resource types for cost optimization scoped to subscriptions, management groups, or the entire tenant. Get an efficiency score for each subscription and can also provide total potential cost saving opportunity. Leverage cost data to understand optimization potential and can also provide different types of safe and secure remediation possibilities . Also it Incorporates the Azure Advisor recommendations.

Key Benefits:
* Cost Optimization
* Maximizes cloud resource utilization by identifying unused or misconfigured resources.
* Cloud Governance
* Safe and Secure Remediation methods

## Resource Type: ApplicationGateway

Application gateways are quite expensive resource in Azure and identifying an Unused Application gateways is going to be an effective cost optimization effort.

Criteria:
*  Application gateways without any backend pool targets configured.
*  Application gateways has unhealthy backend and No success/Direction traffics
*  Resource shouldnt be built recently 


#### AppGw - without Backendpool Targets
```
resources
| join (
    resources
    | where type =~ 'microsoft.network/applicationgateways'
    | mv-expand backendConfig=properties.backendAddressPools
    | summarize TotalBackendCount=count(backendConfig) by name ) on name
| join (
    resources
    | where type =~ 'microsoft.network/applicationgateways'
    | mv-expand Config=properties.backendAddressPools
    | extend emptypoolconfig = Config.properties.backendIPConfigurations
    | where isnull(emptypoolconfig)
    | where Config.properties.backendAddresses == "[]" and properties.redirectConfigurations == "[]"
    | summarize TargetCount=count(Config.properties.backendAddresses) by name ) on name
| where TargetCount == TotalBackendCount
| where type =~ 'Microsoft.Network/applicationGateways'
| where properties.operationalState != 'Stopped'
| extend Details = pack_all()
| project Resource=id, resourceGroup, location, subscriptionId, tags, Details

```

#### Note
If you have identified any Unused AppGW using the above KQL query, need to ensure the same status was persisted for atleast 180 days by running the below Kql in Log Analytics Workspace. 

```
AzureDiagnostics
| where ResourceType == "APPLICATIONGATEWAYS" and OperationName == "ApplicationGatewayAccess"
| where TimeGenerated > ago(180d)
| extend code = case( httpStatus_d between (200 .. 299), "2xx", httpStatus_d between (300 .. 399), "3xx",  httpStatus_d between (400 .. 499), "4xx" , httpStatus_d between (500 .. 599), "5xx", "NULL")
| summarize AggregatedValue = count() by code,  ResourceId, Resource, ResourceGroup, SubscriptionId 

```
Output should not have any 2xx/3xx httpcodes logs reported as below image.

![image](https://github.com/azure-scavengers/Azure-Unused-Resources/blob/b7f220abc8bba624ea56e11ec3d156f5a4dec30b/Docs/AppGwStatus.png)

#### Remediations:

*  Aggressive -  Immediate Decommissioning of the unused application gateway and associated resources (Public IP/Certificates)
*  Conservative - Unused Application gateways can be stopped using the following PS commands which stops the incurring cost to zero.
   
   ```
   $AppGw = Get-AzApplicationGateway -Name "AppGatewayname" -ResourceGroupName "AppgwRSGgroup"
   Stop-AzApplicationGateway -ApplicationGateway $AppGw
   ```

## More Cost saving Resource Types

Assessments mainly focussed on the costly resource types using KQL queries and powershell scripts for the Idle Utilization and misconfigurations.
*  PowerBI Embedded Capacity
*  SQL DBs/Managed Instances (single)
*  SQL DBs (ElasticPools)
*  CosmosDB

On demand Cost optimizing automations using azure functions are available based on resource Utilization.

<b>[Looking for a Demo? Connect Us!](https://airtable.com/appSMMDDdyPWKvAaC/shrrdqfA5X775v2gq)</b>

## Azure Workbook Offerings

Multiple KQL queries have been added into a <b>[Workbook](https://github.com/azure-scavengers/Azure-Unused-Resources.git)</b> to capture all the unused resources runnings on your cloud Infrastructure. You can review the criteria and remediation recommendations for each resource type.

This Azure workbook designed to showcase the Unused or Idle resources running on your environment, also shows the provisioned SKU/Capacities and current state of the resources with the complete lists. You can download the inventory from this workbook and take necessary remediation accordingly. [Analysis Method and Remediation recommendations](https://github.com/azure-scavengers/Azure-Unused-Resources/blob/3d5b34a428e8c6133a76f2594751ceb6d312a10b/Sample-KQLs/Idle-resources-SampleKQLs.md) 
