<a name="browse-with-azure-resource-graph"></a>
# Browse with Azure Resource Graph

If you aren’t familiar Azure Resource Graph, it’s a new service which provides a query-able caching layer over ARM.
This gives us the capability to sort, filter, and search server side which is a vast improvement on what we have today.

<a name="browse-with-azure-resource-graph-benefits-to-tracking-resources-with-azure-resource-graph"></a>
## Benefits to Tracking Resources with Azure Resource Graph

There are compelling reasons to use the Azure Resource Graph for your tracked resources.

- Performance
    - The general performance of using Azure Resource Graph is greatly improved as ARG is at heart an indexing service for your ARM resources. This includes paging and server-side filtering support.
- Query-able
    - Allows for customized query options to provide rich data to customers and allows the browse blade to provide a richer experience to the customer.
- Summary Views
    - The nature of ARG as an indexing and query-able layer allows the browse blade to provide rich summary views to give the customer rich visual representations of their resources.
- Future
    - The ARG browse path will be receiving almost all the new features moving forward while the ARM browse path will only be receiving maintenance work moving forward.

<a name="browse-with-azure-resource-graph-start-with-arm-browse-by-defining-the-asset-type"></a>
## Start with ARM Browse by Defining the Asset Type

Start by creating an asset type definition in PDL with the `Browse` node that has the `Type` set to `ResourceType` and define the `ResourceType` using the appropriate fully qualified resource type name and API version (these values should come from your Resource Provider team):

    ```xml
    <AssetType Name="MyAsset" Text="{Resource MyAsset.text, Module=ClientResources}" ...>
      <Browse Type="ResourceType" />
      <ResourceType
          ResourceTypeName="Resource.Provider/types"
          ApiVersion="2017-04-01" />
    </AssetType>
    ```

Next, update the display name for our asset type to include singular/plural and uppercase/lowercase variants. These will help ensure we display the right text at the right time throughout the portal.

    ```xml
    <AssetType
        Name="MyAsset"
        SingularDisplayName="{Resource MyAsset.singular, Module=ClientResources}"
        PluralDisplayName="{Resource MyAsset.plural, Module=ClientResources}"
        LowerSingularDisplayName="{Resource MyAsset.lowerSingular, Module=ClientResources}"
        LowerPluralDisplayName="{Resource MyAsset.lowerPlural, Module=ClientResources}"
        ...>
      <Browse Type="ResourceType" />
      <ResourceType
          ResourceTypeName="Resource.Provider/types"
          ApiVersion="2017-04-01" />
    </AssetType>
    ```

<a name="browse-with-azure-resource-graph-start-with-arm-browse-by-defining-the-asset-type-moving-forward"></a>
### Moving Forward

What that does mean though is we won’t be loading extensions to gather the extra, supplemental, column data. Instead that will all be served via ARG.

Due to which there the following required from extension authors to onboard.

- Define the columns which you wish to expose
- Craft the query to power your data set
  - To craft the query you can use the in portal advanced query editor [Azure Resource Graph Explorer](https://rc.portal.azure.com#blade/hubsextension/argqueryblade)
  - Ensure the query projects all the framework and extension expected columns
- Onboard your given asset via PDL
  - If you haven't created an Asset follow the previous documentation on how to do that
- Choose how to expose the ARG experience

**Note:** the below contains the PDL, Columns definitions, and Query required to match to an existing AppServices browse experience.

<a name="browse-with-azure-resource-graph-onboarding-an-asset-to-arg"></a>
## Onboarding an asset to ARG

Firstly you'll need to craft a KQL query which represents all possible data for your desired browse view, this includes the required framework columns.

<a name="browse-with-azure-resource-graph-onboarding-an-asset-to-arg-expected-framework-columns"></a>
### Expected Framework columns

| Display name | Expected Column Name | PDL Reference |
| ------------ | -------------------- | ------------- |
| Name | name | N/A - Injected as the first column |
| Resource Id | id | FxColumns.ResourceId |
| Subscription | N/A | FxColumns.Subscription |
| SubscriptionId | subscriptionId | FxColumns.SubscriptionId |
| Resource Group | resourceGroup | FxColumns.ResourceGroup |
| Resource Group Id | N/A | FxColumns.ResourceGroupId |
| Location | location | FxColumns.Location |
| Location Id | N/A | FxColumns.LocationId |
| Resource Type | N/A | FxColumns.ResourceType |
| Type | type | FxColumns.AssetType |
| Kind | kind | FxColumns.Kind |
| Tags | tags | FxColumns.Tags |

<a name="browse-with-azure-resource-graph-kql-query"></a>
## KQL Query

For those who are not familiar with KQL you can use the public documentation as reference. https://docs.microsoft.com/en-us/azure/kusto/query/

Given the framework columns are required we can use the below as a starting point.

1. Go to the [Azure Resource Graph explorer](https://rc.portal.azure.com#blade/hubsextension/argqueryblade)
1. Copy and paste the below query
1. Update the `where` filter to your desire type

```kql
where type =~ 'microsoft.web/sites'
| project name,resourceGroup,kind,location,id,type,subscriptionId,tags
```

That query is the bare minimum required to populate ARG browse.

As you decide to expose more columns you can do so by using the logic available via the KQL language to `extend` and then `project` them in the query.
One common ask is to convert ARM property values to user friendly display strings, the best practice to do that is to use the `case` statement in combination
with extending the resulting property to a given column name.

In the below example we're using a `case` statement to rename the `state` property to a user friendly display string under the column `status`.
We're then including that column in our final project statement. We can then replace those display strings with client references once we migrate it over to
PDL in our extension providing localized display strings.

```kql
where type =~ 'microsoft.web/sites'
| extend state = tolower(properties.state)
| extend status = case(
state == 'stopped',
'Stopped',
state == 'running',
'Running',
'Other')
| project name,resourceGroup,kind,location,id,type,subscriptionId,tags
, status
```

As an example the below query can be used to replicate the 'App Services' ARM based browse experience in ARG.

```kql
where type =~ 'microsoft.web/sites'
| extend appServicePlan = extract('serverfarms/([^/]+)', 1, tostring(properties.serverFarmId))
| extend appServicePlanId = properties.serverFarmId
| extend state = tolower(properties.state)
| extend sku = tolower(properties.sku)
| extend pricingTier = case(
sku == 'free',
'Free',
sku == 'shared',
'Shared',
sku == 'dynamic',
'Dynamic',
sku == 'isolated',
'Isolated',
sku == 'premiumv2',
'PremiumV2',
sku == 'premium',
'Premium',
'Standard')
| extend status = case(
state == 'stopped',
'Stopped',
state == 'running',
'Running',
'Other')
| extend appType = case(
kind contains 'botapp',
'Bot Service',
kind contains 'api',
'Api App',
kind contains 'functionapp',
'Function App',
'Web App')
| project name,resourceGroup,kind,location,id,type,subscriptionId,tags
, appServicePlanId, pricingTier, status, appType
```

<a name="browse-with-azure-resource-graph-pdl-definition"></a>
## PDL Definition

In your extension you'll have a `<Asset>` tag declared in PDL which represents your ARM resource. In order to enable Azure Resource Graph (ARG) support for that asset we'll need to update the `<Browse>` tag to include a reference to the `Query`, `DefaultColumns`, and custom column meta data - if you have any.

<a name="browse-with-azure-resource-graph-pdl-definition-query-for-pdl"></a>
### Query for PDL

Create a new file, we'll use `AppServiceQuery.kml`, and save your query in it.
You can update any display strings with references to resource files using following syntax `'{{Resource name, Module=ClientResources}}'`.

The following is an example using the resource reference syntax.

```kql
where type == 'microsoft.web/sites'
| extend appServicePlanId = properties.serverFarmId
| extend state = tolower(properties.state)
| extend sku = tolower(properties.sku)
| extend pricingTier = case(
    sku == 'free',
    '{{Resource pricingTier.free, Module=BrowseResources}}',
    sku == 'shared',
    '{{Resource pricingTier.shared, Module=BrowseResources}}',
    sku == 'dynamic',
    '{{Resource pricingTier.dynamic, Module=BrowseResources}}',
    sku == 'isolated',
    '{{Resource pricingTier.isolated, Module=BrowseResources}}',
    sku == 'premiumv2',
    '{{Resource pricingTier.premiumv2, Module=BrowseResources}}',
    sku == 'premium',
    '{{Resource pricingTier.premium, Module=BrowseResources}}',
    '{{Resource pricingTier.standard, Module=BrowseResources}}')
| extend status = case(
    state == 'stopped',
    '{{Resource status.stopped, Module=BrowseResources}}',
    state == 'running',
    '{{Resource status.running, Module=BrowseResources}}',
    '{{Resource status.other, Module=BrowseResources}}')
| extend appType = case(
    kind contains 'botapp',
    '{{Resource appType.botapp, Module=BrowseResources}}',
    kind contains 'api',
    '{{Resource appType.api, Module=BrowseResources}}',
    kind contains 'functionapp',
    '{{Resource appType.functionapp, Module=BrowseResources}}',
    '{{Resource appType.webapp, Module=BrowseResources}}')
| project name,resourceGroup,kind,location,id,type,subscriptionId,tags
, appServicePlanId, pricingTier, status, appType
```

<a name="browse-with-azure-resource-graph-pdl-definition-custom-columns"></a>
### Custom columns

To define a custom column you will need to create a `Column` tag in PDL within your `Browse` tag.

A column tag has 5 properties.

```xml
<Column Name="status"
      DisplayName="{Resource Columns.status, Module=ClientResources}"
      Description="{Resource Columns.statusDescription, Module=ClientResources}"
      Format="String"
      WidthInPixels="90" />
```

- Name: The identifier which is used to uniquely refer to your column
- DisplayName: A display string, __this has to be a reference to a resource__
- Description: A description string, __this has to be a reference to a resource__
- Format: See below table for possible format
- WidthInPixels: String, which represents the default width of the column in pixels (e.g. "120")

| Format option | Description |
| ------------- | ----------- |
| String | String rendering of your column |
| Resource | If the returned column is a ARM resource id, this column format will render the cell as the resources name and a link to the respective blade |

<a name="browse-with-azure-resource-graph-pdl-definition-default-columns"></a>
### Default columns

To specify default columns you need to declare a property `DefaultColumns` on your `Browse` `PDL` tag.
Default columns is a comma separated list of column names, a mix of custom columns and framework defined columns from the earlier table. All framework columns are prefixed with `FxColumns.`.

For example `DefaultColumns="status, appType, appServicePlanId, FxColumns.Location"`.

<a name="browse-with-azure-resource-graph-pdl-definition-full-asset-definition"></a>
### Full Asset definition

In the above query example there are 4 custom columns, the below Asset `PDL` declares the custom column meta data which each map to a column in the query above.

It also declares the default columns and their ordering for what a new user of the browse experience should see.

```xml
<AssetType>
    <Browse
        Query="{Query File=./AppServiceQuery.kml}"
        DefaultColumns="status, appType, appServicePlanId, FxColumns.Location">
            <Column Name="status"
                  DisplayName="{Resource Columns.status, Module=ClientResources}"
                  Description="{Resource Columns.statusDescription, Module=ClientResources}"
                  Format="String"
                  WidthInPixels="90" />
            <Column Name="appType"
                  DisplayName="{Resource Columns.appType, Module=ClientResources}"
                  Description="{Resource Columns.appTypeDescription, Module=ClientResources}"
                  Format="String"
                  WidthInPixels="90" />
            <Column Name="appServicePlanId"
                  DisplayName="{Resource Columns.appServicePlanId, Module=ClientResources}"
                  Description="{Resource Columns.appServicePlanIdDescription, Module=ClientResources}"
                  Format="Resource"
                  WidthInPixels="90" />
            <Column Name="pricingTier"
                  DisplayName="{Resource Columns.pricingTier, Module=ClientResources}"
                  Description="{Resource Columns.pricingTierDescription, Module=ClientResources}"
                  Format="String"
                  WidthInPixels="90" />
    </Browse>
</AssetType>
```

<a name="browse-with-azure-resource-graph-releasing-the-azure-resource-graph-experience"></a>
## Releasing the Azure Resource Graph experience

Ensure the SDK version that you use is `5.0.302.27001` or later.

Per Asset you can configure extension side feature flags to control the release of your assets Azure Resource Graph browse experience.

Within your extension config, either hosting service or self hosted, you will need to specify config for your assets with one of the following:

```json
{
    "argbrowseoptions": {
        "YOUR_ASSET_NAME": "OPTION_FROM_THE_TABLE_BELOW",
    }
}
```

| Option | Definition |
| ------ | ---------- |
| AllowOptIn | Allows users to opt in/out of the new experience but will default to the old experience. This will show a 'Try preview' button on the old browse blade and an 'Opt out of preview' button on the ARG browse blade. |
| ForceOptIn | Allows users to opt in/out of the new experience but will default to the new experience. This will show a 'Try preview' button on the old browse blade and an 'Opt out of preview' button on the ARG browse blade |
| Force | This will force users to the new experience. There wil be no 'Opt out of preview' button on the ARG browse blade |
| Disable | This will force users to the old experience. This is the default experience if not flags are set. There wil be no 'Try preview' button on the ARG browse blade |

To test each variation or to test when side loading you can use:

`https://portal.azure.com/?ExtensionName_argbrowseoptions={"assetName":"OPTION"}`

<a name="browse-with-azure-resource-graph-column-summaries-for-extension-provided-columns"></a>
## Column Summaries for Extension-provided Columns

There are summary views available on the browse blade out of the box for certain FX built-in columns to show a map (location-based columns), bar chart, donut chart and list (grid) such a location, resource group, type and subscription. These summary visualizations give customers a quick break-down of their resources by these properties in an easy to understand chart with counts to quickly reason over their resources.

In addition, we are also adding extension-provided columns for browse for a specific resource type. These are based on the `Column` definitions in the PDL. The code uses the `Format` as a hint to provide an appropriate summarization based on a simple count per value in that column. This will work for the vast majority of cases where the column is a string, location, number, datetime, etc. but may not be appropriate for all columns and we have added some customization points to prevent a column from being used in the summary views or to provide custom queries. As well, the PDL allows to specification about which views should be available for the column (map, bar chart, donut chart and/or grid).

<a name="browse-with-azure-resource-graph-column-summaries-for-extension-provided-columns-preventing-the-column-summary"></a>
### Preventing the Column Summary

There are cases where a column is simply not useful as a summary. The `Column` can be marked with the `PreventSummary` property:

```xml
      <Column Name="someColumn"
              DisplayName="{Resource Columns.someColumn, Module=ClientResources}"
              Description="{Resource Columns.someColumn, Module=ClientResources}"
              Format="string"
              Width="80fr"
              PreventSummary="true" />
```

In addition, all columns with the `Format` of `BladeLink` are excluded from summaries.

<a name="browse-with-azure-resource-graph-column-summaries-for-extension-provided-columns-specifying-which-visualizations-to-show-for-column-summary"></a>
### Specifying Which Visualizations to Show for Column Summary

There are cases where the default visualizations for location-based columns (map, bar chart, donut chart and grid) or non-location-based columns (bar chart, donut chart and grid) are not desirable. The `Column` can be marked with the `SummaryVisualizations` property:

```xml
      <Column Name="someColumn"
              DisplayName="{Resource Columns.someColumn, Module=ClientResources}"
              Description="{Resource Columns.someColumn, Module=ClientResources}"
              Format="string"
              Width="80fr"
              SummaryVisualizations="DonutChart,Grid" />
```

These are the available values (can be combined using comma as shown above):

|Visualization|Definition|
| ------ | ---------- |
|Map|Shows a map representation with clickable pins - only valid for location-based columns|
|BarChart|Shows a bar chart with clickable columns|
|DonutChart|Shows a donut chart with clickable sections|
|Grid|Shows a grid with clickable rows - should always be included|
|**Default**|This is the default of every visualization except Map - useful for non-location-based columns|
|**DefaultWithMap**|The is the default of every visualization including Map - useful for location-based columns|


The order of the visualizations does not matter and will not change the order of the items in the drop down in browse.

<a name="browse-with-azure-resource-graph-column-summaries-for-extension-provided-columns-custom-column-handling-for-summary-views"></a>
### Custom Column Handling for Summary Views

If you have a column which doesn't map well to a straight `count() of column` summarization, you can provide queries to change the summarization for the columns. If the summarization is based on an existing column (has a `Column` value), only the `summaryQuery` property needs to be set on the `Column`:

```xml
      <Column Name="someColumn"
              DisplayName="{Resource Columns.someColumn, Module=ClientResources}"
              Description="{Resource Columns.someColumn, Module=ClientResources}"
              Format="string"
              Width="80fr"
              SummaryQuery="{Query File=./QueryForSomeColumnSummary.kml" />
```

Then the query file would include only the summarize portion of the query:

```
summarize someColumnCount=count() by bin(someColumn, 100)
```

The result of the count summarization must be the column name with `Count` appended (`someColumn` column must have `someColumnCount` count).

If a new column must be generated for the summarize, however, then the `columnQueryForSummary` property must point to a query which will produce (extend) that new column separately from the `summaryQuery` because the summary view drill down blade will use that portion of the query to provide the list of resources that match the clicked value:

```xml
      <Column Name="someColumn"
              DisplayName="{Resource Columns.someColumn, Module=ClientResources}"
              Description="{Resource Columns.someColumn, Module=ClientResources}"
              Format="string"
              Width="80fr"
              ColumnQueryForSummary="{Query File=./SomeColumnQueryForSummary.kml}"
              SummaryQuery="{Query File=./SomeColumnSummaryQuery.kml}"
              SummaryColumn="someColumnSummaryColumn" />
```

Then the column query file would include only the extend portion of the query:

```
extend someColumnSummaryColumn = case(
    (someColumn < 50), 'low',
    (someColumn <= 100), 'normal',
    'above')
```

Then the summary query file would include only the summarize portion of the query:

```
summarize someColumnSummaryColumnCount=count() by someColumnSummaryColumn
```

The `SummaryColumn` property needs to be set to the name of the produced (extended) column and again, the count summarization must be the summary column name with `Count` appended (`SummaryColumn` is `someColumnSummaryColumn` and the count must be `someColumnSummaryColumnCount`).

<a name="browse-with-azure-resource-graph-column-summaries-for-extension-provided-columns-dealing-with-non-scalar-values-in-summary"></a>
### Dealing with Non-scalar Values in Summary

The Kusto `summarize` operator requires that the summarize `by` clauses must be scalar and sometimes additional processing needs to be done (like having case-insensitive summary). The way this is handled for non-customized columns is by extending a temporary column casted to a scalar, used in by the summarize and then projected away.

The column query file is often not needed and the summary query file would handle the casting:

```
extend _someColumn = tolower(tostring(someColumn))
| summarize someColumn=any(someColumn),someColumnCount=count() by _someColumn
| project-away _someColumn
```

In this query, first a temporary column (`_someColumn`) is produced with the conversion to scalar (`tostring()`) and then converted to lower case for case-insensitive summarization (`tolower()`). Once the column is projected, the original value must be projected (`someColumn=any(someColumn)`) along with the count (`someColumnCount=count()`) otherwise the value column will be missing from the results. The `by` clause uses the temporary projected column `_someColumn`. Lastly, to prevent the additional temporary column from being returned, the _someColumn is projected away (`project-away _someColumn`).

<a name="browse-with-azure-resource-graph-extensible-commanding-for-arg-browse"></a>
## Extensible commanding for ARG browse

Extensible commanding enables you to author and own your resource-specific commands that allow users to manage their resources at scale with minimal efforts.
Once you have onboarded to ARG browse experience, you can start authoring commands for your browse experience using Typescript. Currently, extension authors can statically define commands associated to their <AssetType>.
*Extensible commanding is only supported for asset types which are onboarded to ARG browse.* The typescript decorator for command takes metadata required for creation and execution of the commands.

Extension authors can specify two sets of commands for their asset type. The generic commands that do not require resource selection (these will be enabled by default) and selection based commands that require resource selection (These will only be enabled if user has selected resources in the browse grid). The generic commands will be placed between `Add` and `Edit columns` command in the toolbar. The selection based commands will be displayed after `Assign tags` command in the toolbar.

Simply specify a command kind and intellisense will prompt you for all the required properties for that command type.
These are the currently supported command types:

```typescript

import { SvgType } from "Fx/Images";

export = MsPortalFxForAsset;

module MsPortalFxForAsset {
    export module ForAsset {
        export module Commands {
            /**
             * Command kinds.
             */
            export const enum CommandKind {
                /**
                 * Kind for the open blade commands.
                 */
                OpenBladeCommand,

                /**
                 * Kind for the menu command.
                 */
                MenuCommand,
            }

            /**
             * Selection Command kinds.
             */
            export const enum SelectionCommandKind {
                /**
                 * Kind for the open blade commands that require selection.
                 */
                OpenBladeSelectionCommand,

                /**
                 * Kind for the ARM commands.
                 */
                ArmCommand,

                /**
                 * Kind for the selection based menu command.
                 */
                MenuSelectionCommand,
            }

            /**
             * Defines the options that are passed to the command decorator.
             */
            export interface CommandOptions {
                /**
                 * The asset type that the commands are associated with.
                 */
                readonly assetType: string;

                /**
                 * The list of commands which do no require resource selection.
                 */
                readonly commands?: ReadonlyArray<Command>;

                /**
                 * The list of commands which require selection.
                 */
                readonly selectionCommands?: ReadonlyArray<SelectionCommand>;
            }

            /**
             * Constrains the @ForAsset.Commands decorator so that it can be applied only to classes implementing 'Contract'.
             */
            export interface Contract {
            }

            /**
             * Constrains the @Commands decorator so that it can be applied only to classes implementing 'Contract'.
             */
            export interface CommandsClass {
                new(...args: any[]): Contract;
            }

            /**
             * Decorator for Asset commands
             * @param options command options
             */
            export function Decorator(options?: CommandOptions) {
                return function (commandsClass: CommandsClass) {
                };
            }

            /**
             * The blade reference options for open blade command.
             */
            export interface BladeReference {
                /**
                 * The blade name.
                 */
                readonly blade: string;

                /**
                 * The extension name for the blade
                 */
                readonly extension?: string;

                /**
                 * The flag indicating whether blade needs to be opened as a context pane.
                 * Defaults to false.
                 */
                readonly inContextPane?: boolean;

                /**
                 * The blade parameters.
                 *
                 * NOTE: Blades that require list of resourceIds in the parameters, should specify {resourceIds} as the parameter value.
                 * Fx will replace the {resourceIds} value with currently selected resource Ids at runtime.
                 */
                readonly parameters?: ReadonlyStringMap<any>;
            }

            /**
             * The marketplace blade reference
             */
            export interface MarketplaceBladeReference {
                /**
                 * The marketplaceItemId to open a create flow.
                 */
                readonly marketplaceItemId?: string;
            }

            /**
             * Interface for Open blade commands.
             */
            export interface OpenBladeCommand extends CommonCommandBase<CommandKind.OpenBladeCommand> {
                /**
                 * The blade reference.
                 * Either a reference to the blade or the marketpkace item id which opens the create flow needs to be specified.
                 */
                readonly bladeReference: BladeReference | MarketplaceBladeReference;
            }

            /**
             * The interface for resource selection for commands.
             */
            export interface RequiresSelection {
                /**
                 * The resource selection for commands.
                 * Default selection is max allowed selection supported by browse grid.
                 */
                readonly selection?: Selection;
            }

            /**
             * The interface for command execution confirmation options.
             */
            export interface ConfirmationOptions {
                /**
                 * The confirmation dialog title to show before execution of the command.
                 */
                readonly title: string;

                /**
                 * The confirmation dialog message to show before execution of the bulk command.
                 */
                readonly message: string;

                /**
                 * The confirmation text input.
                 * User needs to enter this text in order to confirm command execution.
                 */
                readonly explicitConfirmationText?: string;
            }

            /**
             * The interface for commands that require user confirmation.
             */
            export interface ConfirmationCommandBase {
                /**
                 * The command execution confirmation options.
                 */
                readonly confirmation: ConfirmationOptions;
            }

            /**
             * The interface for ARM command definition.
             */
            export interface ArmCommandDefinition {
                /**
                 * Http method POST/DELETE/PATCH etc. By default POST will be used.
                 */
                readonly httpMethodType?: string;

                /**
                 * ARM uri for the command operation.
                 * Uri should be a relative uri with the fixed format - {resourceid}/optionalOperationName?api-version.
                 * Example: "{resourceid}?api-version=2018-09-01-preview
                 */
                readonly uri: string;

                /**
                 * ARM command operation can be long running operation. asyncOperation property specifies how to poll the status for completion of long running operation.
                 */
                readonly asyncOperation?: AsyncOperationOptions;

                /**
                 * Optional list of resource-specific ARM error codes that should be retried for HttpStatusCode.BadRequest(400).
                 *
                 * By default, Fx retries below codes:
                 *     Retry for transient errors with Http status codes: HttpStatusCode.InternalServerError(500), HttpStatusCode.BadGateway(502), HttpStatusCode.ServiceUnavailable(503), HttpStatusCode.GatewayTimeout(504)
                 *     Retry for ARM conflict/throttle errors with status codes: HttpStatusCode.TooManyRequests(409), HttpStatusCode.Conflict(429)
                 * In addition to these, there could be resource-specific errors that need to be retried for HttpStatusCode.BadRequest(400).
                 * If this list is specified, Fx will parse ARM error codes for HttpStatusCode.BadRequest(400) requests and retry in addition to above retries.
                 *
                 * Example: ["PublicIpAddressCannotBeDeleted", "InuseNetworkSecurityGroupCannotBeDeleted"]
                 */
                readonly retryableArmCodes?: ReadonlyArray<string>;

                /**
                 * Optional list of resource-specific ARM error codes that shouldn't be retried.
                 * This helps optimize network calls and improve bulk operation performance.
                 *
                 * By default, Fx won't issue retry for below code regardless of HTTP status code:
                 *    "ScopeLocked"
                 * In addition to this Arm error code, there could be resource-specific error codes that shouldn't be retried.
                 * If this list is specified, Fx will ignore the above mentioned list and only honor this list of Arm codes that shouldn't be retried.
                 *
                 * Example: ["ScopeLocked"]
                 */
                readonly nonRetryableArmCodes?: ReadonlyArray<string>;
            }

            /**
             * Optional Arm command configs to describe how long running ARM operations needs to be polled and results processed.
             */
            export interface AsyncOperationOptions {
                /**
                 * By default when http Accepted (202) status code is received, the Location header will be looked up for polling uri to get the status of long running operation.
                 * A different response header can be specified with the pollingHeaderOverride value.
                 */
                readonly pollingHeaderOverride?: string;

                /**
                 * A property path to look for status in the response body.
                 * By default 'status' property will be looked up to see if it has "Succeeded", "Failed", "InProgress" or "Canceled".
                 */
                readonly statusPath?: string;
            }

            /**
             * The interface for ARM commands.
             * These commands honor default selection which is FullPage.
             */
            export interface ArmCommand extends CommonCommandBase<SelectionCommandKind.ArmCommand>, ConfirmationCommandBase {
                /**
                 * The map of ARM bulk command definitions per resource type.
                 *
                 * NOTE: A command may delete multiple types of resources e.g. browse for merged resource types.
                 * In such cases, ARM command definition can be specified for each resource type.
                 */
                readonly definitions: ReadonlyStringMap<ArmCommandDefinition>;

                /**
                 * The flag indicating whether to launch Fx bulk delete confirmation blade for delete operations.
                 */
                readonly isDelete?: boolean;
            }

            /**
             * The interface for open blade commands that require resource selection.
             */
            export interface OpenBladeSelectionCommand extends CommonCommandBase<SelectionCommandKind.OpenBladeSelectionCommand>, RequiresSelection {
                /**
                 * The blade reference.
                 */
                readonly bladeReference: BladeReference;
            }

            /**
             * The interface for selection based menu command.
             */
            export interface MenuSelectionCommand extends CommonCommandBase<SelectionCommandKind.MenuSelectionCommand>, RequiresSelection {
                /**
                 * The list of commands.
                 */
                readonly commands: ReadonlyArray<OpenBladeSelectionCommand | ArmCommand>; //If a new command type is supported in future, it'll be added to this list depending on whether it needs to be supported in the menu list.
            }

            /**
             * The interface for menu command.
             */
            export interface MenuCommand extends CommonCommandBase<CommandKind.MenuCommand> {
                /**
                 * The list of commands.
                 */
                readonly commands: ReadonlyArray<OpenBladeCommand>;
            }

            /**
             * The interface for commands that require resource selection.
             */
            export type SelectionCommand = OpenBladeSelectionCommand | ArmCommand | MenuSelectionCommand;

            /**
             * The interface for command.
             */
            export type Command = OpenBladeCommand | MenuCommand;

            /**
             * The interface for command selection.
             */
            export interface Selection {
                /**
                 * The max number of selected resources supported by the command operation.
                 */
                readonly maxSelectedItems: number;

                /**
                 * The message shown when user tries to select more than supported items by the command operation.
                 */
                readonly disabledMessage: string;
            }

            /**
             * The interface for common command properties.
             */
            export interface CommonCommandBase<TKind extends CommandKind | SelectionCommandKind> {
                /**
                 * The command kind.
                 */
                readonly kind: TKind;

                /**
                 * The command Id.
                 */
                readonly id: string;

                /**
                 * The command label.
                 */
                readonly label: string;

                /**
                 * The command icon.
                 */
                readonly icon: (
                    {
                        /**
                         * URI to the image element.
                         */
                        path: string;
                    } | {
                        /**
                         * References a built-in SVG element.
                         */
                        image: SvgType;
                    });

                /**
                 * The command tooltip.
                 */
                readonly tooltip?: string;

                /**
                 * The command aria label.
                 */
                readonly ariaLabel?: string;
            }
        }
    }
}


```

Here is a sample of defining various asset commands, represented by a single TypeScript file in your extension project.

```typescript

import { ForAsset } from "Fx/Assets/Decorators";
import * as ClientResources from "ClientResources";
import { AssetTypeNames } from "_generated/ExtensionDefinition";
import { SvgType } from "Fx/Images";

@ForAsset.Commands.Decorator({
    assetType: AssetTypeNames.virtualServer, // Asset type name associated with commands
    commands: [ // Generic commands that do not require resource selection
        {
            kind: ForAsset.Commands.CommandKind.OpenBladeCommand,
            id: "OpenBladeCommandId",  // Unique identifier used for controlling visibility of commands
            label: ClientResources.AssetCommands.openBlade,
            ariaLabel: ClientResources.AssetCommands.openBlade,
            icon: {
                image: SvgType.AddTile,
            },
            bladeReference: {
                blade: "SimpleTemplateBlade",
                extension: "SamplesExtension", // An optional extension name, however, must be provided when opening a blade from a different extension
                inContextPane: true, // An optional property to open the pane in context pane
            },
        },
        {
            kind: ForAsset.Commands.CommandKind.OpenBladeCommand,
            id: "OpenCreateCommandId",
            label: ClientResources.AssetCommands.openCreate,
            icon: {
                image: SvgType.Cubes,
            },
            bladeReference: {
                marketplaceItemId: "Microsoft.EngineV3", // Opens marketplace create flow
            },
        },
    ],
    selectionCommands: [ // Commands that require resource selection
        {
            kind: ForAsset.Commands.SelectionCommandKind.ArmCommand,  // Executes ARM bulk operations
            id: "BulkDelete",
            label: ClientResources.AssetCommands.delete,
            icon: {
                image: SvgType.Delete,
            },
            definitions: {
                "microsoft.test/virtualservers": {
                    httpMethodType: "DELETE",
                    uri: "{resourceid}?api-version=2018-09-01-preview", // The fixed format that starts with {resourceid}
                    asyncOperation: {
                        pollingHeaderOverride: "Azure-AsyncOperation",
                    },
                },
            },
            isDelete: true,  // Launches the default bulk delete confirmation blade provided by Fx on user click
            confirmation: {
                title: ClientResources.AssetCommands.confirmDeleteTitle,
                message: ClientResources.AssetCommands.confirmDeleteMessage,
            },
        },
        {
            kind: ForAsset.Commands.SelectionCommandKind.ArmCommand,  // Executes ARM bulk operations
            id: "BulkStart",
            label: ClientResources.AssetCommands.start,
            icon: {
                image: SvgType.Start,
            },
            definitions: {
                "microsoft.test/virtualservers": {
                    uri: "{resourceid}?api-version=2018-09-01-preview", // The fixed format that starts with {resourceid}
                },
            },
            confirmation: {
                title: ClientResources.AssetCommands.confirmStartTitle,
                message: ClientResources.AssetCommands.confirmStartMessage,
            },
        },
        {
            kind: ForAsset.Commands.SelectionCommandKind.MenuSelectionCommand,
            id: "SelectionBasedMenuCommand",
            label: ClientResources.AssetCommands.menuCommand,
            icon: {
                image: SvgType.PolyResourceLinked,
            },
            selection: {
                maxSelectedItems: 2,
                disabledMessage: ClientResources.AssetCommands.disabledMessage,
            },
            commands: [
                {
                    kind: ForAsset.Commands.SelectionCommandKind.OpenBladeSelectionCommand,
                    id: "SelectionBasedOpenBladeCommand",
                    label: ClientResources.AssetCommands.openBlade1,
                    icon: {
                        image: SvgType.AzureQuickstart,
                    },
                    bladeReference: {
                        blade: "SimpleTemplateBlade",
                    },
                },
                {
                    kind: ForAsset.Commands.SelectionCommandKind.OpenBladeSelectionCommand,
                    id: "SelectionBasedOpenBladeCommand",
                    label: ClientResources.AssetCommands.openBlade2,
                    icon: {
                        image: SvgType.QuickStartPoly,
                    },
                    bladeReference: {
                        blade: "DiTemplateBlade",
                    },
                },
            ],
        },
    ],
})
export class VirtualServerCommands {
}


```

<a name="browse-with-azure-resource-graph-extensible-commanding-for-arg-browse-how-to-hide-your-asset-commands-in-different-environments"></a>
### How to hide your asset commands in different environments

You can control visibility of individual or all your commands in different environments by setting the hideAssetTypeCommands extension feature flag in your config.
You can specify a comma separated list of asset command ids or "*" to hide all the extensible commands on your browse blade

If you’re using the hosting service, you can do this by updating the relevant environment configuration file (e.g. portal.azure.cn.json file)

```json
    {
        "hideAssetTypeCommands": {
          "YOUR_ASSETTYPE_NAME_DEFINED_IN_PDL": ["YOUR_COMMAND_ID_TO_HIDE"],
          "YOUR_ASSETTYPE_NAME_DEFINED_IN_PDL": ["YOUR_COMMAND_ID1_TO_HIDE", "YOUR_COMMAND_ID2_TO_HIDE"],
          "YOUR_THIRD_ASSETTYPE_NAME_DEFINED_IN_PDL": ["*"]
        }
    }
```

<a name="browse-with-azure-resource-graph-extensible-commanding-for-arg-browse-how-to-hide-your-asset-commands-in-different-environments-testing-hiding-commands-locally"></a>
#### Testing hiding commands locally

For the desired environment append the following feature flags.

```txt
    ?microsoft_azure_myextension_hideassettypecmmands={"MyAsset":["MyCommandId1", "MyCommandId2"]}
```

If you want to test hiding all your commands, you can specify ["*"].

```txt
    ?microsoft_azure_myextension_hideassettypecmmands={"MyAsset":["*"]}
```
