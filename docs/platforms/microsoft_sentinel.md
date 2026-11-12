- [x] Search feature
- [x] Export feature
    * [x] Remove detection rules
    * [x] Disable detection rules
    * [x] Near real-time (NRT) detection rules
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

???+ warning

    On Microsoft Sentinel, `search_auth` is used for both searches and exports: `export_auth` is not read.

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
| export_auth        | Yes       | String        | Authentication method: "app" or "default". Not read on Microsoft Sentinel, see the note above. |
| query_period       | Yes       | Integer       | The lookback of a scheduled rule, in hours: how far back each run searches. See [schedule](#schedule). |
| query_frequency    | Yes       | Integer       | How often a scheduled rule runs, in hours. See [schedule](#schedule).       |
| days_ago           | Yes       | Integer       | The lookback period of time when searching, in days. |
| timeout            | Yes       | Integer       | Search timeout in seconds.                   |
| alert_prefix       | No        | String        | Prefix for the exported alert.                                              |
| threshold_operator | No        | String        | Operator used for alert threshold comparison. Accepted values can be found [here](https://learn.microsoft.com/en-us/rest/api/securityinsights/alert-rules/create-or-update?view=rest-securityinsights-2024-03-01&tabs=HTTP#triggeroperator). |
| threshold_value    | No        | Integer       | Threshold value for generating alerts.                                      |
| suppress_status    | No        | Boolean       | Whether to stop running the rule for `suppress_period` after it triggers. See [suppression](#suppression). |
| suppress_period    | No        | Integer       | How long (in hours) the rule stops running after it triggers. Required when `suppress_status` is true. Microsoft Sentinel wants a duration even with suppression off: droid then sends this value, or 5 hours when it is not set. |
| incident_status    | No        | Boolean       | Whether to create incidents for triggered alerts.                           |
| grouping_status    | No        | Boolean       | Enable or disable grouping of alerts into incidents. Grouping only merges alerts of the same rule. |
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
query_period = 2
threshold_operator = "GreaterThan"
threshold_value = 0
suppress_status = false
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

### Schedule

A rule is deployed as a scheduled rule, unless it asks to be a near real-time rule.

**Scheduled rules** run every `query_frequency` hours and search the last `query_period` hours of data. An alert is raised when the number of results compares to `threshold_value` as `threshold_operator` says: `GreaterThan` and `0` raise an alert on the first result. A rule overrides both values with `custom.query_frequency` and `custom.query_period`.

Logs reach the workspace some minutes after they happen, more for some sources. With a `query_period` equal to `query_frequency`, an event that arrives after the run covering it never matches: a run only looks at its own window. A `query_period` longer than `query_frequency`, such as 2 and 1, catches it on the next run. The overlap has a cost: an event already ingested in time matches both runs, and raises two alerts. Incident grouping puts the second alert in the incident of the first, as long as the incident is still open and the entities match.

**Near real-time (NRT) rules** run about every minute on the data ingested since their last run, so they neither miss late events nor raise duplicates. A rule is deployed as NRT with `query_period: 0`, the value which also makes it near real-time on [Microsoft XDR](microsoft_xdr.md#schedule):

```yaml
...
custom:
  query_period: 0
```

An NRT rule has no frequency, lookback or threshold: `query_frequency`, `query_period`, `threshold_operator` and `threshold_value` do not apply, and every result raises an alert. Suppression, incident creation, grouping and entity mappings apply as for a scheduled rule.

???+ warning

    Microsoft Sentinel allows 50 NRT rules per workspace. Keep them for the rules where minutes matter, such as a sign-in with a break-glass account.

???+ note

    A rule already deployed can switch between scheduled and NRT: droid deletes the deployed rule and creates it again with its new kind, as Microsoft Sentinel cannot be relied on to change it in place. If the creation fails after the deletion, droid logs that the rule is no longer deployed in that workspace. The integrity check (`droid rules integrity`) reports a rule deployed with the wrong kind.

### Suppression

Suppression does not deduplicate alerts: once the rule raises an alert, Microsoft Sentinel stops running it for `suppress_period` hours. Any activity in that window is never searched, not even a different entity doing the same thing. Prefer incident grouping to reduce noise, and keep `suppress_status` for rules that need it.

### MSSP mode

The MSSP mode (`--mssp`) in Microsoft Sentinel allows to perform various operations involving multiple workspaces.

When using `droid rules search`, this mode allows to query multiple Microsoft Sentinel workspaces for one or multiple rules. To achieve that, `droid` will first list all the Microsoft Sentinel workspaces using an Azure Resource Graph query and will query all the workspaces with parallel tasks.

This mode also allows to export (`droid rules export`) detection rules to specific Microsoft Sentinel workspaces included in the droid configuration file.

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

    The integrity feature (`droid rules integrity`) is also available.

Each entry also accepts a `variants` key, which selects the telemetry sources that workspace carries when a log source is served by more than one. See [log source variants](../log_source_variants.md#per-customer-telemetry).

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
| query_period | integer       | Lookup data from the last X hour, or `0` for a near real-time rule, see [schedule](#schedule) |
| grouping_period | integer    | Group the alerts created within X hours, overriding `grouping_period` |
| grouping_matching_entities | list | Group the alerts by these entity types only, for instance `["Account"]`. Sets the grouping method to `Selected` |
| entity_mappings | dict       | Configure the entity mappings  |
| sentinel_entities | list     | Configure the sentinel entity mappings  |

Use the following example to configure the entity mappings and sentinel entity mappings in a Sigma rule:

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
  sentinel_entities:
    - fieldA
    - fieldB
```