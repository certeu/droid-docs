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
--rules rules/sigma/ \
--convert -d
```

???+ info

    If you want to use the [Sigma filters](https://sigmahq.io/docs/meta/filters.html), you can store your filters in a directory and use the `sigma_filters_directory` parameter. See the [configuration](./configuration.md#configure-the-basics).

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

## MSSP

Currently the [MSSP mode](./platforms/microsoft_sentinel.md#mssp-mode) is available for Microsoft Sentinel and Microsoft XDR.

```bash
droid -cf droid_config.toml --platform microsoft_sentinel --rules sigma/sigmahq-core/windows/process_creation/proc_creation_win_wmic_susp_process_creation.yml --search --mssp
```

When using the MSSP mode, you have the ability to apply specific filters for designated customers. By adding `customer_name` and `customers_filters_directory` in the TOML configuration in  `platforms.<platform>.export_list_mssp.CustomerName`, `droid` will add the default filters directory from `base` AND the Sigma filters in `customer_filters_directory`.

Example:

```
            [platforms.microsoft_xdr.export_list_mssp.Zoidberg]

            tenant_id = "122d2a69-c233-4824-a009-a431d839d799"
            customer_name = "Zoidberg"
            customer_filters_directory = "filters/zoidberg/"
```

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
