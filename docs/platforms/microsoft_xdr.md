- [x] Search feature
- [x] Export feature
    * [x] Remove detection rules
    * [x] Disable detection rules
- [ ] MSSP feature
    * [ ] Search in multiple tenants
    * [ ] Search in multiple tenants
- [x] Detection rule actions
    * [x] forceUserPasswordReset
    * [x] disableUser
    * [x] markUserAsCompromised
    * [x] stopAndQuarantineFile
    * [x] restrictAppExecution
    * [x] initiateInvestigation
    * [x] runAntivirusScan
    * [x] collectInvestigationPackage
    * [x] isolateDevice
    * [x] blockFile
    * [x] allowFile

### Limitations

Since the PySigma backend [microsoft365defender](https://github.com/AttackIQ/pySigma-backend-microsoft365defender) does not yet support correlation rules only standard sigma and raw rules are supported.

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

???+ note

    When using the default authentication, set the `tenant_id` in the configuration

### Supported Platforms

Both Sigma and raw rules are supported under the platform name "microsoft365defender".

### Main config

| Parameter    | Mandatory | Default Value | Description   |
| ------------ | --------- | ------------- | -------------- |
| search_auth  | Yes       | N/A           | Authentication method: "app" or "default" |
| export_auth  | Yes       | N/A           | Authentication method: "app" or "default" |
| query_period | Yes       | N/A           | The period of time the query should run over. Accepted Values are: 1h, 3h, 12h, 24h or 0 for near realtime, however this only works on quick queries, Sigma queries should always be OK, but RAW rules might not work |
| alert_prefix | No        | None         | Prefix for the exported rule name  |
| tenant_id | No        | None         | Tenant ID when using "default" |


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

An identifier is required for all actions. However, the script will automatically add the necessary fields to the query for actions with only one selection.

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

???+ danger

    Never add both "block" and "allow" files simultaneously, as this can cause significant issues.

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

This field is needed for asset mapping in the resulting alerts. It defaults to Impacted Asset = deviceId, but that might not always work.

| Impacted Asset Type | Identifier (Allowed Values)                                                                                                         |
| ------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| Device              | deviceId, deviceName                                                                                                                |
| Mailbox             | accountUpn, initiatingProcessAccountUpn                                                                                             |
| User                | targetAccountUpn, accountObjectId, accountSid, accountUpn, initiatingProcessAccountObjectId, initiatingProcessAccountSid, initiatingProcessAccountUpn |

```yaml
custom:
    # ...
    impactedAssets:
    - impactedAssetType: Device
        identifier: deviceId #deviceId, deviceName
    - impactedAssetType: Mailbox
        identifier: initiatingProcessAccountUpn #accountUpn, initiatingProcessAccountUpn
    - impactedAssetType: User
        identifier: initiatingProcessAccountUpn # targetAccountUpn, accountObjectId, accountSid, accountUpn, initiatingProcessAccountObjectId, initiatingProcessAccountSid, initiatingProcessAccountUpn
```