# Platforms

The following platforms are supported for the search and export features:

- Splunk
- Microsoft Sentinel
- Microsoft Defender for Endpoint (only through Microsoft Sentinel for now)
- Elastic Security

???+ info

    Additional platform support will be added in future updates.

## Splunk

To use Splunk, the following sections are required:

- `[platforms.splunk]`: handle the basic configuration and the SIEM settings for searches and alerts
- `[platforms.splunk.savedsearch_parameters]`: saved search parameters
- `[platforms.splunk.action]`: action for rules

### Environment variables

`droid` will require the following environment variables to authenticate on Splunk:

- `DROID_SPLUNK_USER`: Splunk user
- `DROID_SPLUNK_PASSWORD`: Splunk user password

The following environment variables can be set:

- `DROID_SPLUNK_URL`: Replace the Splunk URL parameter.
- `DROID_SPLUNK_WEBHOOK_URL`: Replace the Splunk webhook URL

### Main config

```toml
[platforms.splunk]

url = "splunksh-dev.pizza-planet.local" # (1)!
verify_cert = true
port = "8089"

# user and password are passed in environment variable

test_earliest_time = "-24h@h" # (2)!
test_latest_time = "now" # When using option (--search/-s) # (3)!
job_ttl = 86400 # (4)!
acl_update_owner = "nobody" # (5)!
acl_update_perms_read = "group1, group2" # (6)!

## For alerts
# General config

earliest_time = "-1h@h" # (7)!
latest_time = "now" # (8)!
cron_schedule = "0 * * * *"
alert_expiration = "7d"
...
```

1.  The hostname of Splunk, can be replace by an env variable.

2.  Search feature (--search/-s): earliest time for the searches. It must be a Splunk time modifier.

3.  Search feature (--search/-s): latest time for the searches. It must be a Splunk time modifier.

4.  Search feature (--search/-s): Set the job TTL in seconds.

5.  Search feature (--search/-s): Set the owner of the search. Useful if droid reports for findings.

6.  Search feature (--search/-s): Set the groups read permissions.

7.  Export (deploy) feature (--export/-e): earliest time for the saved-searches. It must be a Splunk time modifier.

8.  Export (deploy) feature (--export/-e): latest time for the saved-searches. It must be a Splunk time modifier.

???+ info

    DROID fetches the url of Splunk via the key `url` but this value can be replaced with the environment `DROID_SPLUNK_URL`. This is also valid for webhook URL when `DROID_SPLUNK_WEBHOOK_URL` exists.

### Savedsearch config


```toml
[platforms.splunk.savedsearch_parameters]

alert_type = "number of events"
app = "pizza_app_rules"
sharing = "app"
alert_comparator = "greater than"
alert_threshold = 0
"alert.track" = 1
allow_skew = "67%"
"alert.suppress" = 1
"alert.digest_mode" = 0 # Per result (per row)
#"alert.suppress.period" = "8h" # Optional
#"alert.suppress.fields" = "pizza_client,host" # Optional

    # [platforms.splunk.savedsearch_parameters.suppress_fields_groups.group_name]
    # Optional: define a suppress fields groups by logsource (.e.g. web)

    #[platforms.splunk.savedsearch_parameters.suppress_fields_groups.windows_image_load]

    #category = "image_load"
    #product = "windows"
```

### Action

Using emails:

```toml
[platforms.splunk.action]

actions = "email"
"action.email.to" = "bender.rodriguez@pizza-planet.local"
"action.email.subject" = "Alert: $name$"
"action.email.message.alert" = "Description: $description$"
```

Using webhooks:

```toml
[platforms.splunk.action]

actions = "webhook"
"action.webhook.param.url" = "https://automation.pizza-planet.local/2342/32322" # (1)!
```

1.  Place your webhook URL or replace it in your env variable.

## Microsoft Sentinel

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

The keys `workspace_id` and `workspace_name` are the base workspace declaration but this values can be replaced with the environments `DROID_AZURE_WORKSPACE_ID` and `DROID_AZURE_WORKSPACE_NAME`.

## Microsoft Defender for Endpoint

???+ warning

    There are currently limitations with Microsoft Defender for Endpoint (MDE). The full integration of the search and export feature is pending but using this platform is possible through Microsoft Sentinel. Just use the argument `-sm`.

```toml
[platforms.microsoft_defender]

## For searches

days_ago = 30
timeout = 120 # Search timeout for azure

## Authentication

search_auth = "default"
export_auth = "default"
```

## Elastic Security

### Limitations

The Raw EQL Backend does not support Index detection.
So in cases of RAW EQL rules it is necessary to specify the indexes in the rule itself, or it will be defaulted to logs-*
This limitation does not apply to Raw ES|QL rules since there the index is always specified in the rule itself.


### Authentication

Right now only basic authentication is supported.
If API Tokens are prefered, open a feature request.

- `basic`: Authenticate using username and password

When using the Azure Registation App, load the following environment variables:

- `DROID_ELASTIC_USERNAME` : Elastic username
- `DROID_ELASTIC_PASSWORD` : Elastic password

### Supported Platforms
For Sigma Rules both esql (ES|QL) and eql (EQL) are supported.
However since only esql supports correlations rules, it is advised to use esql only.
There are no known benefits to using EQL over ES|QL.

For Raw Rules, both esql (ES|QL) and eql (EQL) are supported.
Always use the `raw_language` custom field to specify the language used in the Sigma yaml file.

For non correlation ES|QL rules, the metadata fields are automatically added to the query for better deduplication by the Elastic backend.
For correlation ES|QL rules this is not needed since correlation rules are already deduplicated by their nature.

### Main config

| Parameter              | Mandatory | Default Value | Description                                                                                                 |
| ---------------------- | --------- | ------------- | ----------------------------------------------------------------------------------------------------------- |
| kibana_url             | Yes       | N/A           | The Base URL to your Kibana, since the Security API used goes via Kibana not Elasticsearch                  |
| kibana_ca              | No        | False         | Certificate Chain used on the Kibana host                                                                   |
| elastic_ca             | No        | None          | Certificate Chain used on the Elastic host, can also be the same as Kibana, only used when Searching        |
| elastic_tls_verify     | No        | False         | Used to turn on and off Certificate Validation for Elastic Connections, only used when Searching            |
| schedule_interval      | No        | "1"           | Interval at which the alert rule should run                                                                 |
| schedule_interval_unit | No        | "h"           | Interval unit minute (m) or hour (h)                                                                        |
| license                | No        | "DRL"         | The license of your rule                                                                                    |
| legacy_esql            | No        | false         | This flag is only if you are on an Elastic Version that needs the square brackets around the metadata info. |
| building_block_prefix  | No        | None          | Optional prefix for Building Block Rules to distinguish from Alert rules, defaults to BB                    |
| alert_prefix           | No        | BB            | Optional prefix to your imported rules to distinguish them                                                  |
```toml
[platforms]

[platforms.elastic]
auth_method = "basic"
kibana_url = "https://kibana.test.org"
kibana_ca = "kibana-ca.pem" # Or False to ignore certificates
elastic_ca = "elastic-ca.pem"
elastic_tls_verify = true
elastic_hosts = [
    "https://elasticsearch01.test.org:9200",
    "https://elasticsearch02.test.org:9200",
    "https://elasticsearch03.test.org:9200",
]
schedule_interval = 5
schedule_interval_unit = "m"
license = "DRL"
alert_prefix = "SIGMA"
legacy_esql = false

[platforms.elastic.pipelines.windows_process_creation]
pipelines = ["pipelines/ecs_pipeline.yml"]
product = "windows"
category = "process_creation"
...
```

### Sigma Custom Fields

The Custom Fields available with the Elastic Integration are:
| Custom Field   | Values          | Description                                                                        |
| -------------- | --------------- | ---------------------------------------------------------------------------------- |
| raw_language   | esql/eql        | The Language used on raw Rules                                                     |
| disabled       | true/false      | Set to true if you want to disable the rule                                        |
| removed        | true/false      | Set to true if you want to delete the rule                                         |
| building_block | true/false      | Set to true if your rule should be a building block                                |
| index          | list of strings | Set one or more indexes to be used. Use a list (-) if you want more than one index |




## Microsoft XDR

### Limitations

Since the pySigma converter does not yet support correlation rules only standard sigma and raw rules are supported.


### Authentication

Only App registration is supported.

When using the Azure Registation App, load the following environment variables:

- `app`: Azure Registration App

When using the Azure Registation App, load the following environment variables:

- `DROID_AZURE_TENANT_ID`: Azure tenant ID
- `DROID_AZURE_CLIENT_ID`: Client ID of the registration app
- `DROID_AZURE_CLIENT_SECRET`: Azure client secret

### Supported Platforms

Both Sigma and Raw rules are supported under the plattform name "microsoft365defender"

### Main config

| Parameter    | Mandatory | Default Value | Description                                                                                                                                                                                                           |
| ------------ | --------- | ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| search_auth  | Yes       | N/A           | How you want to authenticate for Search queries. Right now only App Regs are supported                                                                                                                                |
| export_auth  | Yes       | N/A           | How you want to authenticate for exporting queries. Right now only App Regs are supported                                                                                                                             |
| query_period | Yes       | N/A           | The period of time the query should run over. Accepted Values are: 1h, 3h, 12h, 24h or 0 for near realtime, however this only works on quick queries, Sigma queries should always be OK, but RAW rules might not work |
| alert_prefix | No        | False         | Prefix for the exported rule name                                                                                                                                                                                     |



```toml
[platforms]
[platforms.microsoft365defender]

search_auth = "app"
export_auth = "app"
query_period = "1h"
alert_prefix = "SIGMA"

[platforms.microsoft365defender.pipelines.process_creation]
pipelines = ["microsoft_365_defender"]
product = "windows"
category = "process_creation"

...
```

### Sigma Custom Fields

The Custom Fields available with the Elastic Integration are:
| Custom Field   | Values     | Description                                                                                   |
| -------------- | ---------- | --------------------------------------------------------------------------------------------- |
| disabled       | true/false | Set to true if you want to disable the rule                                                   |
| removed        | true/false | Set to true if you want to delete the rule                                                    |
| action         | list       | The actions that should be performed on rule trigger, see action section for more information |
| impactedAssets | list       | The impactedAssets are needed for Asset mapping to the alerts. Defaults to Device = deviceId  |

### Action

The following actions are supported.
The identifier is needed for all actions, however the script will automatically add the necessary fields to the query for the actions that only have one selection.
Important, never add block and allow file at the same time, the world will burn!

| Action                      | Identifier                                                   | Isolation Type  | Device Group          |
| --------------------------- | ------------------------------------------------------------ | --------------- | --------------------- |
| forceUserPasswordReset      | accountSid, initiatingProcessAccountSid                      |                 |                       |
| disableUser                 | accountSid, initiatingProcessAccountSid                      |                 |                       |
| markUserAsCompromised       | accountObjectId, initiatingProcessAccountObjectId            |                 |                       |
| stopAndQuarantineFile       | sha1, initiatingProcessSHA1                                  |                 |                       |
| restrictAppExecution        |                                                              |                 |                       |
| initiateInvestigation       |                                                              |                 |                       |
| runAntivirusScan            |                                                              |                 |                       |
| collectInvestigationPackage |                                                              |                 |                       |
| isolateDevice               |                                                              | selective, full |                       |
| blockFile                   | sha256, sha1, initiatingProcessSHA1, initiatingProcessSHA256 |                 | List of Device Groups |
| allowFile                   | sha256, sha1, initiatingProcessSHA1, initiatingProcessSHA256 |                 | List of Device Groups |


```yaml
custom:
  disabled: false
  actions:
    - action: forceUserPasswordReset
      identifier: accountSid #accountSid, initiatingProcessAccountSid
    - action: disableUser
      identifier: initiatingProcessAccountSid #accountSid, initiatingProcessAccountSid
    - action: markUserAsCompromised
      identifier: initiatingProcessAccountObjectId #accountObjectId, initiatingProcessAccountObjectId
    - action: stopAndQuarantineFile
      identifier: initiatingProcessSHA1 # sha1, initiatingProcessSHA1
    - action: restrictAppExecution
    - action: initiateInvestigation
    - action: runAntivirusScan
    - action: collectInvestigationPackage
    - action: isolateDevice
      isolationType: selective # full, selective

## Allow or Block file, never both!
    - action: blockFile
      deviceGroupNames:
      identifier: initiatingProcessSHA256 # sha256, sha1, initiatingProcessSHA1, initiatingProcessSHA256
    #- action: allowFile
    #  deviceGroupNames:
    #  identifier: initiatingProcessSHA256 # sha256, sha1, initiatingProcessSHA1, initiatingProcessSHA256

```


### Impacted Assets

This field is needed for asset mapping in the resulting alerts.
It defaults to Impacted Asset = deviceId, but that might not always work

| Impacted Asset Type | Identifier (Allowed Values)                                                                                                         |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Device              | deviceId, deviceName                                                                                                                |
| Mailbox             | accountUpn, initiatingProcessAccountUpn                                                                                             |
| User                | targetAccountUpn, accountObjectId, accountSid, accountUpn, initiatingProcessAccountObjectId, initiatingProcessAccountSid, initiatingProcessAccountUpn |


```yaml
  impactedAssets:
    - impactedAssetType: Device
      identifier: deviceId #deviceId, deviceName
    - impactedAssetType: Mailbox
      identifier: initiatingProcessAccountUpn #accountUpn, initiatingProcessAccountUpn
    - impactedAssetType: User
      identifier: initiatingProcessAccountUpn # targetAccountUpn, accountObjectId, accountSid, accountUpn, initiatingProcessAccountObjectId, initiatingProcessAccountSid, initiatingProcessAccountUpn
```