# Log source variants

A Sigma log source names the *kind* of telemetry a rule needs, not where that telemetry comes from. `windows/process_creation` usually means Sysmon `EventCode 1`, but the same detection is just as valid over the process events of a third-party EDR. `webserver` usually means the Azure WAF logs, but some workspaces carry `AGWAccessLogs` instead.

A **variant** is one telemetry source serving one log source. Each variant has its own pipeline group, produces its own query, and is deployed as its own object on the platform.

???+ info

    Variants are entirely optional. A log source served by a single pipeline group needs no `variant` key at all, and every configuration written before this feature existed keeps behaving exactly as it did.

## Declaring variants

Variants are declared by adding `variant` to the pipeline groups that serve the same log source, and by marking exactly one of them `primary`.

```toml title="droid_config.toml" hl_lines="6 7 13"
[platforms.splunk.pipelines.windows_process_creation]

pipelines = ["splunk_windows", "pipelines/splunk_process_creation.yml"]
product = "windows"
category = "process_creation"
variant = "sysmon" # (1)!
primary = true # (2)!

[platforms.splunk.pipelines.windows_process_creation_edr]

pipelines = ["splunk_windows", "pipelines/splunk_process_creation_edr.yml"]
product = "windows"
category = "process_creation"
variant = "edr" # (3)!
```

1.  The name of the telemetry source. It is used to derive the rule identity and to select the variant per platform or per customer.

2.  Exactly one group per log source must claim the primary role. It keeps the bare Sigma id and the bare rule title.

3.  No `primary` key, so this group deploys under a derived identity.

Rules matching `windows/process_creation` now convert twice, once through each pipeline group. The Sysmon pipeline points at the Sysmon index, the EDR pipeline points at the EDR one, and both queries are deployed.

### Configuration errors

Once a log source is served by more than one group, `droid` refuses to guess and raises an error when:

- one of the groups has no `variant` name
- two groups declare the same `variant` name
- no group, or more than one group, claims `primary`

???+ warning

    Before variants existed, two pipeline groups matching the same log source silently resolved to whichever one the configuration happened to list first. That coin flip is now a loud failure.

## Identity on the platform

The primary variant deploys under the **bare Sigma UUID and the bare rule title**, which is exactly what `droid` did before variants existed. Nothing already deployed moves, and no migration is needed.

Every other variant derives a stable identity from the Sigma id and its own name.

| Variant | Microsoft Sentinel and XDR `rule_id` | Splunk saved search name |
| ------- | ------------------------------------ | ------------------------ |
| primary | the Sigma `id` | `Suspicious process` |
| `edr` | `uuid5(sigma_id, "edr")` | `Suspicious process [edr]` |

The derived UUID is a function of the Sigma id and the variant name only, so it is stable across runs and unique per rule. Two different rules never collide on the same variant name.

## Platform-wide telemetry

A single-tenant deployment that only carries some of the declared sources can narrow them once, on the platform itself.

```toml title="droid_config.toml"
[platforms.splunk]

variants = ["edr"]
```

Only those variants are ever converted or deployed. Omit the key to get all of them, which is the default. It narrows exactly what a per-customer list narrows, as described [below](#what-an-allowlist-does-and-does-not-narrow).

## Per-customer telemetry

Customers rarely carry every source. In [MSSP mode](./usage.md#mssp), each `export_list_mssp` entry may declare the variants that customer actually has.

```toml title="droid_config.toml" hl_lines="6 12"
[platforms.microsoft_xdr.export_list_mssp.Zoidberg]

tenant_id = "122d2a69-c233-4824-a009-a431d839d799"
customer_name = "Zoidberg"
customer_filters_directory = "filters/zoidberg/"
variants = ["defender", "edr"]

[platforms.microsoft_xdr.export_list_mssp.Slurm]

tenant_id = "0e3a2b3f-9c26-4b91-9f7e-30c30e9bb7a1"
customer_name = "Slurm"
variants = ["edr"]
```

Slurm is never sent the native Defender query, which would run against data they do not have and would silently return nothing forever.

An entry **without** a `variants` key receives every variant, which is what configurations predating this feature expect.

### What an allowlist does and does not narrow

One `variants` list covers the whole platform, so it can only narrow the log sources that actually offer a choice. A pipeline group is subject to the allowlists **only if it declares a `variant` name**.

Nearly every log source is served by a single group with no `variant` key. Those rules keep deploying to everyone regardless of the allowlist. Were it otherwise, adding `variants` to one customer to pick their `process_creation` source would silently stop every other rule in the repository from reaching them.

???+ tip

    That also gives you the escape hatch for the opposite case. To withhold a single-source log source from a customer, give its pipeline group a `variant` name and leave that name out of their list.

You can preview the result before exporting anything. `droid rules convert --mssp` renders the same narrowing, so what you see per customer is what will be deployed to them.

```bash
droid --debug rules convert \
--rules rules/rules/sigma/ \
--config-file droid_config.toml \
--platform microsoft_xdr \
--mssp
```

???+ note

    Run the command with `--debug` to see the `[customer / variant]` labels. Without it the queries are printed unlabelled.

### Orphan reporting

Removing a variant from a customer's `variants` list stops `droid` deploying it, but whatever was already pushed keeps running in their tenant or workspace. When `droid` skips a customer for that reason it looks the rule up, and warns if it is still there.

```
Orphan rule foo.yml (defender) still deployed in tenant <id> for 'Slurm',
which no longer carries that telemetry. Remove it manually if it is no longer wanted.
```

`droid` never deletes it. Only the operator knows whether the leftover is stale or still wanted, and a detection deleted by surprise is worse than one that is merely reported.

???+ warning

    `droid` can only report a variant it still knows about. Deleting a pipeline group from the configuration entirely leaves `droid` with no name to look up. Remove the variant from every customer's `variants` list first, let a run report the leftovers, and only then drop the group.

## Splunk

Splunk has no MSSP concept. Customers are scoped inside the query itself, by the `index` and `splunk_server` conditions the pipeline injects. A variant is therefore **one saved search**, with every relevant customer index merged in its pipeline. The number of saved searches is `rules x variants`, independent of how many customers there are.

```yaml title="splunk_process_creation_edr.yml" hl_lines="8"
name: Splunk EDR process creation
priority: 100

transformations:
  - id: index_condition
    type: add_condition
    conditions:
      index: edr_zoidberg,edr_slurm
      splunk_server: "*prod.planet-express.local"
    rule_conditions:
      - type: logsource
        category: process_creation
        product: windows
```

### Suppression fields

Suppression fields are named per log source, but variants have different schemas. Suppressing a Sysmon rule on `Computer` says nothing about the same detection over EDR data, so a variant group may declare its own.

```toml title="droid_config.toml" hl_lines="7"
[platforms.splunk.pipelines.windows_process_creation_edr]

pipelines = ["splunk_windows", "pipelines/splunk_process_creation_edr.yml"]
product = "windows"
category = "process_creation"
variant = "edr"
"alert.suppress.fields" = "agent_id,process_path"
```

This takes precedence over the log source wide [`savedsearch_parameters.suppress_fields_groups`](./platforms/splunk.md#savedsearch-config) entry. A variant that shares the primary's schema simply omits the key and inherits it.

??? info "Resolution semantics"

    For a given rule, the effective suppression fields are resolved as follows:

    - the rule's own `custom: alert.suppress.fields` field, if set
    - else the `alert.suppress.fields` of the variant's pipeline group, if set
    - else the matching `savedsearch_parameters.suppress_fields_groups` entry, if any
    - else the platform wide `savedsearch_parameters."alert.suppress.fields"`, if set
