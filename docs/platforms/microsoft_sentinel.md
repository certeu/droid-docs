- [x] Search feature
- [x] Export feature
    * [x] Remove detection rules
    * [x] Disable detection rules
- [x] MSSP feature
    * [x] Search and retrieve results from multiple tenants
    * [x] Export in multiple tenants
- [ ] Detection rule actions

???+ note

    It is possible to deploy Microsoft Defender for Endpoint rules to Sentinel using the the `sm` argument.

### Authentication

Two authentication mode are supported:

- `default`: Default authentication via `az login`
- `app`: Azure Registration App

When using the default authentication:

1. Head to `portal.azure.com` and authenticate using MFA
2. Use `az cli` to fetch the authentication session

When using the Azure Registation App, load the following environment variables:

- `DROID_AZURE_TENANT_ID`: Azure tenant ID
- `DROID_AZURE_CLIENT_ID`: Client ID of the registration app
- `DROID_AZURE_CLIENT_SECRET`: Azure client secret

The keys `workspace_id`, `workspace_name`, `subscription_id` and `resource_group` are the base workspace declaration but they can be replaced with the following environment variables:

- `DROID_AZURE_WORKSPACE_ID`
- `DROID_AZURE_WORKSPACE_NAME`
- `DROID_AZURE_SUBSCRIPTION_ID`
- `DROID_AZURE_RESOURCE_GROUP`

???+ note

    The authentication method can be provided using the environment variables `DROID_AZURE_SEARCH_AUTH` and `DROID_AZURE_EXPORT_AUTH`. This will override the parameters `search_auth` and `export_auth` from the TOML configuration.

**Custom authentication via hook**:

We have also added the ability retrieve Azure Tokens from any custom app via a hook.

If the environment variable `DROID_AZURE_TOKEN_HOOK` is set with a URL, the hook attempts to acquire the token from that URL via a `GET` request. The response should contain a JSON object with the `access_token` key, which will be used as the token.

The hook function  supports dynamic URL configuration using placeholders:

- `{TENANT_ID}`: Replaced with the provided or default tenant ID.
- `{SCOPE}`: Replaced with the requested scope.

Additional infos:

- Uses the cached token if it is still valid and the scope matches.
- Supports retries for token retrieval, with retry count and delay.
- If `DROID_AZURE_TOKEN_X_API_KEY` environment variable is present, it will be included in the request headers.

### Permissions

The required permissions are:

| Role | Type | Description |
| --- | --- | --- |
|  Microsoft Contributor | SecurityInsights | Ability to perform Hunting queries and edit detection rules |
|  Read access   | Azure Resource Graph | MSSP: Read all the resources of any subscription |

### Main config

| Parameter          | Mandatory | Type Value | Description                                                                 |
| ------------------ | --------- | ------------- | --------------------------------------------------------------------------- |
| search_auth        | Yes       | String        | Authentication method: "app" or "default".                                  |
| export_auth        | Yes       | String        | Authentication method: "app" or "default".                                  |
| query_period       | Yes       | Integer       | The period of time the query should run over, in hours.                     |
| query_frequency    | Yes       | Integer       | How frequently the query should run, in hours.                              |
| days_ago           | Yes       | Integer       | The lookback period of time when searching, in days. |
| timeout            | Yes       | Integer       | Search timeout in seconds.                   |
| alert_prefix       | No        | String        | Prefix for the exported alert.                                              |
| threshold_operator | No        | String        | Operator used for alert threshold comparison. Accepted values can be found [here](https://learn.microsoft.com/en-us/rest/api/securityinsights/alert-rules/create-or-update?view=rest-securityinsights-2024-03-01&tabs=HTTP#triggeroperator). |
| threshold_value    | No        | Integer       | Threshold value for generating alerts.                                      |
| suppress_status    | No        | Boolean       | Whether to suppress the alert or not.       |
| suppress_period    | No        | Integer       | Time period (in hours) to suppress the alert after it triggers.             |
| incident_status    | No        | Boolean       | Whether to create incidents for triggered alerts.                           |
| grouping_status    | No        | Boolean       | Enable or disable grouping of alerts into incidents.                        |
| grouping_period    | No        | Integer       | Time window (in hours) for grouping alerts into incidents.                  |
| grouping_reopen    | No        | Boolean       | Whether to reopen closed incidents if new alerts match the group criteria.  |
| grouping_method    | No        | String        | Method for grouping alerts. Accepted values can be found [here](https://learn.microsoft.com/en-us/rest/api/securityinsights/alert-rules/create-or-update?view=rest-securityinsights-2024-03-01&tabs=HTTP#matchingmethod)"  |
| workspace_id       | Yes       | String        | Workspace ID used for search queries.                                       |
| workspace_name     | Yes       | String        | Name of the base workspace.                                                 |
| subscription_id    | Yes       | String        | Subscription ID required for setting alerts.                                |
| resource_group     | Yes       | String        | Resource group name for setting alerts                   |

```toml

[platforms.microsoft_sentinel]

## For searches

days_ago = 30
timeout = 120

## For alerts
alert_prefix = "[PLANET-EXPRESS]"
query_frequency = 1
query_period = 14
threshold_operator = "GreaterThan"
threshold_value = 1
suppress_status = true
suppress_period = 2
incident_status = true
grouping_status = true
grouping_period = 5
grouping_reopen = false
grouping_method = "AllEntities"

## Authentication

search_auth = "app"
export_auth = "app"

workspace_id = "d32b8ab7-650e-4dcb-b576-461827daf5b1"
workspace_name = "sentinel-planet_express-prod"
subscription_id = "2b27a0e0-8e25-46fe-b2a6-3fcaed7cad57"
resource_group = "planet_express_resource_group"

## Pipelines example

[platforms.microsoft_sentinel.pipelines]

    [platforms.microsoft_sentinel.pipelines.windows_process_creation]

    pipelines = ["pipelines/sentinel_process_creation.yml", "azure_monitor"]
    product = "windows"
    category = "process_creation"

    [platforms.microsoft_sentinel.pipelines.windows_file_event]

    pipelines = ["pipelines/sentinel_file_event.yml"]
    product = "windows"
    category = "file_event"

```

### MSSP mode

The MSSP mode (`--mssp`) in Microsoft Sentinel allows to perform various operations involving multiple workspaces.

When searching (`--search`), this mode allows to query multiple Microsoft Sentinel workspaces for one or multiple rules. To achieve that, `droid` will first list all the Microsoft Sentinel workspaces using an Azure Resource Graph query and will query all the workspaces with parallel tasks.

This mode also allows to export (`--export`) detection rules to specific Microsoft Sentinel workspaces included in the droid configuration file.

To designate the Microsoft Sentinel workspaces, define them under the `export_list_mssp` section as a dictionary.

```toml

[platforms.microsoft_sentinel.export_list_mssp]

    [platforms.microsoft_sentinel.export_list_mssp.momcorp]

    workspace_name = "momcorp_sentinel"
    tenant_id = "10e5ca4d-25ac-4d17-ae8d-0314ff0d8d44"
    resource_group_name = "momcorp_resource_group_prod"
    subscription_id = "98b80a67-4fec-424c-8f06-56be08deae77"

    [platforms.microsoft_sentinel.export_list_mssp.doop]

    workspace_name = "doop_sentinel"
    tenant_id = "900b0bdf-4de1-48bb-bda4-862f638a92ac"
    resource_group_name = "doop_resource_group_prod"
    subscription_id = "94406b12-28d7-4505-9bfc-fade6ec9a560"
```

???+ note

    The integrity feature (`--integrity`) is also available.

When using the search mode along with the `--mssp` argument you can exclude some workspaces from the search.

```toml
[platforms.microsoft_sentinel]
...
mssp_search_exclude_list = ["Foo", "Sentinel3"]
```

### Sigma Custom Fields

| Custom Field   | Values     | Description                                                                                   |
| -------------- | ---------- | --------------------------------------------------------------------------------------------- |
| disabled       | true/false | Set to true if you want to disable the rule                                                   |
| removed        | true/false | Set to true if you want to delete the rule                                                    |
| query_frequency| integer    | Run query every X hour |
| query_period | integer       | Lookup data from the last X hour  |
| entity_mappings | dict       | Configure the entity mappings  |

Use the following example to configure the entity mappings in a Sigma rule:

```yaml
...
custom:
  entity_mappings:
    - entity_type: IP
      field_mappings:
        - identifier: Address
          column_name: RemoteIP
    - entity_type: Host
      field_mappings:
        - identifier: HostName
          column_name: DeviceName
```