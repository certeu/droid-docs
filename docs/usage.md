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
droid -cf droid_config.toml --platform microsoft_xdr \
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

    This features works only with the [supported platforms](./platforms/index.md).

???+ info

    `droid` will report for findings as per the configured search timerange in your configuration. If there is any hit, it will raise a **warning** but not an error exit-code.

```bash
droid -cf droid_config.toml --platform microsoft_sentinel \
--rules sigma/sigmahq-core/windows/process_creation/proc_creation_win_wmic_susp_process_creation.yml \
--search
```

You can use the search feature to use Microsoft XDR converted rules with Microsoft Sentinel as a search head. This will use your Microsoft Sentinel setup to search. It's also compatible with the `-mssp` mode.

```bash
droid -cf droid_config.toml --platform microsoft_xdr \
--rules sigma/sigmahq-core/windows/process_creation/proc_creation_win_wmic_susp_process_creation.yml \
--search --sentinel-xdr
```

[MSSP mode](#mssp-mode) for Microsoft Sentinel:

```bash
droid -cf droid_config.toml --platform microsoft_sentinel --rules sigma/sigmahq-core/windows/process_creation/proc_creation_win_wmic_susp_process_creation.yml --search --mssp
```

`droid` will report for any findings.

### MSSP mode

`droid` comes with an MSSP mode (`--mssp`). Currently, this mode allows to query multiple Microsoft Sentinel workspaces for one or multiple rules. To achieve that, `droid` will first list all the Microsoft Sentinel workspaces using an Azure Resource Graph query and will query all the workspaces with parallel tasks.


???+ info

    The MSSP mode requires to have the proper permissions as outlined in the Microsoft Sentinel [permissions section](./platforms/microsoft_sentinel.md#permissions).

## Convert

```bash
droid -cf droid_config.toml --platform splunk \
--rules rules/sigma/ \
--compile -d
```

???+ info

    If you want to use the [Sigma filters](https://sigmahq.io/docs/meta/filters.html), you can store your filters in a directory and use the `sigma_filters_directory` parameter. See the [configuration](./configuration.md#configure-the-basics).

## Test using Atomic Red Team

???+ example

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

    This feature verify if the id, description and rule search match the platform's saved search.

```bash
droid -cf droid_config.toml --platform splunk \
--rules rules/rules/sigma/ \
--integrity -d
```