# Platforms

The following platforms are supported for the search and export features:

- Splunk
- Microsoft Sentinel
- Microsoft Defender for Endpoint (only through Microsoft Sentinel for now)

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



