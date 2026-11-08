???+ warning "Upgrading from droid 0.4.0: Graph API changes"

    Microsoft removed several properties of the custom detection rules API on 2026-10-01. droid now manages the rules on Graph **v1.0** with their replacements. Most rules and configurations keep working unchanged, except:

    | Change | What to do |
    | --- | --- |
    | `custom.impactedAssets` is no longer pushed | Move it to [`custom.xdr_entity_mappings`](#entity-mappings). droid warns about every rule still using it |
    | `isolateDevice` without `isolationType` | The isolation type was silently dropped before, it is now an error: set `full` or `selective` |

    Behaviour changes, without anything to do:

    - Every ATT&CK tactic tag of a rule is pushed, each with the techniques belonging to it, where only the last tactic was kept before. See [MITRE ATT&CK](#mitre-attck).
    - The device groups of `blockFile` and the isolation type of `isolateDevice` are now actually pushed.
    - The first export after the upgrade may update every rule once, to the new properties.
    - New rules get the Sigma rule id as their id on the platform. Rules already deployed keep theirs.
    - A rule Microsoft auto-disabled after repeated failures is left disabled with a warning, and reported by the integrity check.
    - The integrity check compares every setting export deploys (schedule, severity, title, tactics, actions, entity mappings), where it only compared the query, the description and the enabled state. Rules changed on the portal by hand, which passed before, are now reported.

- [x] Search feature
- [x] Export feature
    * [x] Remove detection rules
    * [x] Disable detection rules
- [x] MSSP feature
    * [x] Search in multiple tenants
    * [x] Export in multiple tenants
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

Since the PySigma backend [pySigma-backend-kusto](https://github.com/AttackIQ/pySigma-backend-kusto) does not yet support correlation rules only standard sigma and raw rules are supported.

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

Alternatively you can add a credential file config to the toml file:

```toml
[platforms]
[platforms.microsoft_xdr]
credential_file = "microsoft365defender-authentication.yml"
```

The file should look like this:

```yaml
client_id: 123456
client_secret: 123456
tenant_id: 123456
```

This will take precedence over the environment variables.

For certificate-based authentication, you can add the following configuration:

```toml
[platforms]
[platforms.microsoft_xdr]
auth_cert = "your_cert_file.pem"
```

Replace `your_cert_file.pem` with the path to your certificate file. This method will use the provided certificate for authentication. The certificate password is only needed if the private key. To provide a password you can use the environment variable `DROID_AZURE_CERT_PASS` or use `cert_pass: "password"` in the credential file.

???+ note

    The PEM file specified in the `auth_cert` configuration must contain both the private key and the public key, as the public key is used for fingerprinting purposes.

The keys `workspace_id` and `workspace_name` are the base workspace declaration but this values can be replaced with the environments `DROID_AZURE_WORKSPACE_ID` and `DROID_AZURE_WORKSPACE_NAME`.

???+ note

    When using the default authentication, set the `tenant_id` in the configuration

**Custom authentication via hook**:

We have also added the ability retrieve Azure Tokens from any custom app via a hook.

If the environment variable `DROID_AZURE_TOKEN_HOOK` is set with a URL, the hook attempts to acquire the token from that URL via a `GET` request. The response should contain a JSON object with the `access_token` key, which will be used as the token.

The hook function  supports dynamic URL configuration using placeholders:

- `{TENANT_ID}`: Replaced with the provided or default tenant ID.

Additional infos:

- Uses the cached token if it is still valid and the scope matches.
- Supports retries for token retrieval, with retry count and delay.
- If `DROID_AZURE_TOKEN_X_API_KEY` environment variable is present, it will be included in the request headers.

### Permissions

The required permissions for the app registration are the following:

| Microsoft Graph | Type | Description |
| --- | --- | --- |
|   CustomDetection.ReadWrite.All       | Application | Read and write all custom detection rules |
|   ThreatHunting.Read.All       | Application | Run hunting queries |


### Supported Platforms

Both Sigma and raw rules are supported under the platform name `microsoft_xdr`.

### Main config

| Parameter    | Mandatory | Default Value | Description   |
| ------------ | --------- | ------------- | -------------- |
| search_auth  | Yes       | N/A           | Authentication method: "app" or "default" |
| export_auth  | Yes       | N/A           | Authentication method: "app" or "default" |
| days_ago     | No        | 1             | The lookback period time when using the search feature expressed in days (from 1 to 30) |
| query_period | Yes       | N/A           | How often the rule runs: 1h, 3h, 12h, 24h, or 0 for continuous (near real-time), which only works on quick queries. A rule overrides it with `custom.query_period`, and a group of rules with `query_period_groups` |
| alert_prefix | No        | None          | Prefix for the exported rule name  |
| tenant_id | No        | None         | Tenant ID when using "default" |
|credential_file|No|None|A YAML file with your credentials if you do not want to use the environment variables|
|auth_cert|No|None|A certificate file (PEM) when using the certificate-based authentication|

```toml
[platforms]
[platforms.microsoft_xdr]

search_auth = "app"
export_auth = "app"
query_period = "1h"
days_ago = 3
alert_prefix = "SIGMA"

[platforms.microsoft_xdr.pipelines.process_creation]
pipelines = ["pipelines/xdr_process_creation.yml", "microsoft_xdr"]
product = "windows"
category = "process_creation"
...
```

### MSSP mode

The MSSP mode (`--mssp`) exports the detection rules to every tenant listed under `export_list_mssp`. Each entry also accepts a `variants` key, which selects the telemetry sources that tenant carries when a log source is served by more than one. See [log source variants](../log_source_variants.md#per-customer-telemetry).

```toml
[platforms.microsoft_xdr.export_list_mssp.Zoidberg]

tenant_id = "122d2a69-c233-4824-a009-a431d839d799"
customer_name = "Zoidberg"
customer_filters_directory = "filters/zoidberg/"
```

### Sigma Custom Fields

| Custom Field   | Values     | Description                                                                                   |
| -------------- | ---------- | --------------------------------------------------------------------------------------------- |
| disabled       | true/false | Set to true if you want to disable the rule                                                   |
| removed        | true/false | Set to true if you want to delete the rule                                                    |
| actions        | list       | The actions that should be performed on rule trigger, see action section for more information |
| query_period   | 0/1h/3h/12h/24h | How often this rule runs, overriding the configuration |
| xdr_entity_mappings | list  | Maps the query columns to the entities of the alerts, see the [entity mappings](#entity-mappings) section |
| impactedAssets | list       | **Removed by Microsoft**, ignored with a warning: use `xdr_entity_mappings` |

### Action

An identifier is required for all actions, except the device actions, which always act on `DeviceId`. droid pushes each identifier as the advanced hunting column holding it, so the query must return that column: `initiatingProcessSHA1` needs `InitiatingProcessSHA1`, `sha1` needs `SHA1`, `accountSid` needs `AccountSid`, the device actions need `DeviceId`.

A file action can take both a SHA-1 and a SHA-256 identifier as a list, for instance `identifier: [sha1, sha256]`. `stopAndQuarantineFile` also needs `DeviceId`, the device the file is stopped on, and `isolateDevice` requires an isolation type.

Removing an action from a rule removes it from the platform on the next export. The email actions (soft or hard delete, move to Inbox, Junk or Deleted Items) have no custom field: droid leaves the ones set on the portal alone, and does not report them in the integrity check.

| Action                      | Identifier                                                   | Isolation Type  | Device Group          |
| --------------------------- | ------------------------------------------------------------ | --------------- | --------------------- |
| forceUserPasswordReset      | accountSid, initiatingProcessAccountSid, requestAccountSid, onPremSid |        |                       |
| disableUser                 | accountSid, initiatingProcessAccountSid, requestAccountSid, onPremSid |        |                       |
| markUserAsCompromised       | accountObjectId, initiatingProcessAccountObjectId, servicePrincipalId, recipientObjectId | |              |
| stopAndQuarantineFile       | sha1, initiatingProcessSHA1 (deviceId is always added)       |                 |                       |
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


### MITRE ATT&CK

droid pushes the `attack.<tactic>` and `attack.t<id>` tags of a rule as the tactics of its alerts, sub-techniques grouped under their technique. Microsoft refuses a technique under a tactic it does not belong to, so each technique goes under the tagged tactics it belongs to in MITRE ATT&CK, or under its own tactics when the rule tags none of them. The ATT&CK data comes from pySigma, which downloads it on first use, as for Elastic.

ATT&CK v18 split Defense Evasion into Stealth and Defense Impairment, while Microsoft and Sigma still use Defense Evasion: droid pushes those techniques under Defense Evasion. Should Microsoft still refuse a technique, its ATT&CK being older than droid's, droid deploys the rule without it and warns.

### Entity mappings

The custom field `xdr_entity_mappings` maps the columns the query returns to the entities attached to the alerts, which determine how alerts are grouped into incidents. Without it, droid maps the device: `hosts` with `deviceIdColumn: DeviceId`, so the query must return `DeviceId`. A rule querying a table without it, identity or email events for instance, must set its own mappings.

It replaces `impactedAssets`, removed by Microsoft. It has the shape of the Microsoft Sentinel `entity_mappings`, but the Microsoft XDR vocabulary, which is why it has its own name: a rule can carry both.

| Entity type          | Identifiers                                                                                 |
| -------------------- | ------------------------------------------------------------------------------------------- |
| accounts             | aadUserIdColumn, dnsDomainColumn, nameColumn, ntDomainColumn, sidColumn, upnColumn, upnSuffixColumn |
| amazonResources      | amazonResourceIdColumn                                                                      |
| azureResources       | resourceIdColumn                                                                            |
| cloudApplications    | appIdColumn, nameColumn                                                                     |
| dns                  | domainNameColumn, hostIpAddressColumn, serverIpColumn                                       |
| files                | nameColumn, sha1Column, sha256Column                                                        |
| googleCloudResources | fullResourceNameColumn                                                                      |
| hosts                | deviceIdColumn, dnsDomainColumn, nameColumn, netBiosNameColumn, ntDomainColumn              |
| ips                  | addressColumn, scopeColumn                                                                  |
| mailboxes            | primaryAddressColumn                                                                        |
| mailClusters         | queryColumn, sourceColumn                                                                   |
| mailMessages         | networkMessageIdColumn, recipientColumn, senderColumn, subjectColumn                        |
| oAuthApplications    | oAuthAppIdColumn                                                                            |
| processes            | sha1Column, sha256Column                                                                    |
| registryValues       | keyColumn, valueNameColumn                                                                  |
| securityGroups       | distinguishedNameColumn, objectIdColumn, sidColumn                                          |
| urls                 | addressColumn                                                                               |

Each entry becomes one entity: map the columns identifying the same account, host or file in the same entry.

```yaml
custom:
  xdr_entity_mappings:
    - entity_type: hosts
      field_mappings:
        - identifier: deviceIdColumn
          column_name: DeviceId
        - identifier: nameColumn
          column_name: DeviceName
    - entity_type: accounts
      field_mappings:
        - identifier: sidColumn
          column_name: InitiatingProcessAccountSid
        - identifier: upnColumn
          column_name: InitiatingProcessAccountUpn
```

An unknown entity type or identifier, or an entry without field mappings, fails the export of the rule.

Deleting the field from a rule brings the default device mapping back on the next export.

???+ note

    The columns must be returned by the query. With Sigma rules using `| project`, or with raw rules, check that the mapped columns are part of the output.

???+ note "Migrating from impactedAssets"

    | impactedAssets                                    | xdr_entity_mappings                               |
    | ------------------------------------------------- | ------------------------------------------------- |
    | `Device` / `deviceId`                             | `hosts` / `deviceIdColumn`: `DeviceId`            |
    | `Device` / `deviceName`                           | `hosts` / `nameColumn`: `DeviceName`              |
    | `Mailbox` / `accountUpn`                          | `mailboxes` / `primaryAddressColumn`: `AccountUpn` |
    | `User` / `accountSid`                             | `accounts` / `sidColumn`: `AccountSid`            |
    | `User` / `accountUpn`                             | `accounts` / `upnColumn`: `AccountUpn`            |
    | `User` / `accountObjectId`                        | `accounts` / `aadUserIdColumn`: `AccountObjectId` |

    As `impactedAssets` did, droid maps the device of a rule without entity mappings. Setting `xdr_entity_mappings` replaces that default: include `hosts` in it to keep the device.

### Disabled rules

`custom.disabled` sets the status of the rule. Microsoft may also auto-disable a rule that keeps failing: droid leaves it disabled with a warning rather than re-enabling it on every export, and the integrity check reports it, until it is fixed and re-enabled on the platform.
