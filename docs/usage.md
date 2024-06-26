---
hide:
  - navigation
---

# Usage

`droid` is able to:

- Validate the content of the rules
- Convert the Sigma rules
- Perform ad-hoc search on the platform and report for findings
- Export (deploy) the rules
- Check the integrity of the rules on platforms

???+ note

    `droid` is a command line tool.

    - Make sure to precise the droid configuration file using `-cf`
    - You can select a directory (this will load all the rules within sub-directories) or a single rule
    - You can generate a JSON logging file using `-j`

---

## Validate

???+ note

    This requires a validation configuration, see [here](./configuration.md#configure-the-validation).

```bash
droid -cf droid_config.toml --platform microsoft_defender \
--rules sigma/sigmahq-core/windows/process_creation/proc_creation_win_wmic_susp_process_creation.yml \
--validate
```

## Convert

```bash
droid -cf droid_config.toml --platform splunk \
--rules sigma/sigmahq-core/windows/ \
--compile -d
```

## Search

???+ note

    This features works only with the [supported platforms](./platforms.md).

???+ info

    `droid` will report for findings as per the configured search timerange in your configuration. If there is any hit, it will raise a **warning** but not an error exit-code.

```bash
droid -cf droid_config.toml --platform azure \
--rules sigma/sigmahq-core/windows/process_creation/proc_creation_win_wmic_susp_process_creation.yml \
--search
```

You can use the search feature for Microsoft for Endpoint with Microsoft Sentinel as a search head.

```bash
droid -cf droid_config.toml --platform microsoft_defender \
--rules sigma/sigmahq-core/windows/process_creation/proc_creation_win_wmic_susp_process_creation.yml \
--search --sentinel-mde
```

[MSSP mode](#mssp-mode) for Microsoft Sentinel:

```bash
droid -cf droid_config.toml --platform azure --rules sigma/sigmahq-core/windows/process_creation/proc_creation_win_wmic_susp_process_creation.yml --search --mssp
```

`droid` will report for any findings.

### MSSP mode

`droid` comes with an MSSP mode (`--mssp`). Currently, this mode allows to query multiple Sentinel workspaces for one or multiple Sigma rules. To achieve that, `droid` will first list all the Microsoft Sentinel workspaces using a Azure Graph Query and will query all the workspaces with parallel tasks.


???+ info

    Prerequisites:

    - The Azure account used by Sentinel needs to have the ability to query all the subscriptions resources via Azure Graph Query
    - A graph query listing all the Microsoft Sentinel workspaces resource ids along with the workspace name

???+ note

    You will find in the droid configuration example a pre-cooked query for that.

```toml
...
graph_query = 'resources | where name contains "SecurityInsights" | extend workspaceId = tostring(properties.workspaceResourceId) | project name, workspaceId'
...
```

## Convert

```bash
droid -cf droid_config.toml --platform splunk \
--rules rules/sigma/ \
--compile -d
```

## Test using Atomic Red Team

???+ note

    This feature is under development, stay tuned!

## Export

???+ info

    If you have set one or multiple rule with any of the custom field `disabled` or `removed` as `True`, `droid` will make sure it is set as disabled or remove the detection rule if it exists on the platform.

```bash
droid -cf droid_config.toml --platform splunk \
--rules rules/rules/sigma/ \
--export -d
```

## Integrity

???+ info

    This feature verify if the title, description and the rule search match the platform's saved search.

```bash
droid -cf droid_config.toml --platform splunk \
--rules rules/rules/sigma/ \
--integrity -d
```