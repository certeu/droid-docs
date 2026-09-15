- [ ] Search feature
- [x] Export feature
    * [x] Remove detection rules
    * [x] Disable detection rules
- [x] Correlation rules
- [ ] MSSP feature
- [ ] Raw rules
- [ ] Detection rule actions

The platform name is `harfang_lab`. It relies on the [pySigma-backend-harfanglab](https://pypi.org/project/pySigma-backend-harfanglab/) backend.

???+ info

    Unlike the other supported platforms, the HarfangLab backend is not a query language backend. It emits a transformed Sigma document, which `droid` pushes as-is into the `content` field of the rule. The Sigma engine embedded in HarfangLab is the one evaluating the rule.

### Limitations

**Searching is not supported.** HarfangLab does not expose an endpoint to run a Sigma rule against the collected telemetry, so `droid rules search` is refused for this platform.

**MSSP mode is not supported.** Both `--mssp` and `--search` are rejected before the configuration is loaded, so a misconfigured pipeline fails early rather than half way through an export.

**Raw rules are not supported.** Since the platform stores Sigma documents, a raw rule would carry the very content the Sigma path already produces. Rules are always treated as Sigma rules, even when they sit in the raw rules directory.

**Some Sigma fields are rejected by the backend.** The following fields have no equivalent on the platform and the backend refuses any rule using one of them:

`Hash`, `Hashes`, `Provider_Name`, `SourceCommandLine`, `TargetParentProcessId`

Such a rule is reported as a warning, skipped, and is neither exported nor integrity checked. The run still exits with a zero exit code, in the same way an unsupported correlation type is handled, so that one incompatible rule does not fail a whole CI/CD pipeline.

```
WARNING  Backend does not support one of the fields used by the rule: rules/my_rule.yml -
         error: Rule contains unsupported field 'Hashes' for the HarfangLab Sigma backend!
```

???+ warning

    `Hashes` is a common Sigma field. Expect a share of an upstream ruleset such as SigmaHQ to be skipped for this reason.

### Environment variables

`droid` will require the following environment variable to authenticate on HarfangLab:

- `DROID_HARFANGLAB_TOKEN`: HarfangLab API token

The following environment variables can be set:

- `DROID_HARFANGLAB_URL`: Replace the HarfangLab URL parameter
- `DROID_HARFANGLAB_SOURCE_ID`: Replace the source ID parameter
- `DROID_HARFANGLAB_SOURCE_ID_CORRELATION`: Replace the correlation source ID parameter

???+ danger

    The API token must never be written in the configuration file. Pass it through `DROID_HARFANGLAB_TOKEN`.

### Sources

Detection rules are stored in HarfangLab under a **source**, which acts as the container the rules belong to. `droid` looks up a rule within a single source, so the source dedicated to `droid` should not be managed by hand in parallel.

Correlation rules live in their own collection, backed by their own source. Two sources are therefore needed:

- `source_id`: the source holding the Sigma rules
- `source_id_correlation`: the source holding the correlation rules

`source_id_correlation` is only required if you export correlation rules. If it is missing and a correlation rule is exported, the export fails with an explicit error.

???+ warning

    HarfangLab tells rules apart on their Sigma `id`, not on their name. Two rules may share a name, but a duplicated `id` is refused. Since the lookup is scoped to one source, a rule holding the same `id` in **another** source is invisible to `droid` and still blocks the creation. The error message points this out when it happens.

### Main config

| Parameter               | Mandatory | Default Value | Description   |
| ----------------------- | --------- | ------------- | ------------- |
| url                     | Yes       | N/A           | URL of the HarfangLab instance, port included |
| source_id               | Yes       | N/A           | ID of the source holding the Sigma rules |
| source_id_correlation   | No        | None          | ID of the source holding the correlation rules, required to export correlation rules |
| tls_verify              | No        | true          | Verify the TLS certificate of the instance |
| timeout                 | No        | 120           | Timeout of the API requests in seconds |
| alert_prefix            | No        | None          | Prefix for the exported rule name |
| global_state            | No        | alert         | Default state of the exported rules, see the rule state section |
| hl_status               | No        | testing       | Maturity of the rule on the platform: `experimental`, `stable` or `testing` |
| block_on_agent          | No        | false         | Block the matching activity on the agent |
| quarantine_on_agent     | No        | false         | Quarantine the matching file on the agent |
| user_agent              | No        | A browser user agent | User agent sent with the API requests |

```toml
[platforms.harfang_lab]

url = "https://hurukai.pizza-planet.local:8443" # (1)!
# token is passed in environment variable
source_id = "ebae6f85-e097-4987-8524-4398240e7d9a" # (2)!
source_id_correlation = "35d913b3-f575-4d23-93f7-b2e78c0bfbea" # (3)!
tls_verify = true
timeout = 120

alert_prefix = "SIGMA"

## Default state applied to the exported rules

global_state = "alert" # (4)!
hl_status = "testing" # experimental|stable|testing
block_on_agent = false
quarantine_on_agent = false

[platforms.harfang_lab.pipelines.windows_process_creation]

pipelines = ["harfanglab"]
product = "windows"
category = "process_creation"
```

1.  The hostname of the HarfangLab instance, can be replaced by `DROID_HARFANGLAB_URL`.

2.  Sigma rules are exported to this source, can be replaced by `DROID_HARFANGLAB_SOURCE_ID`.

3.  Correlation rules are exported to this source, can be replaced by `DROID_HARFANGLAB_SOURCE_ID_CORRELATION`.

4.  One of `alert`, `block`, `disabled` or `quarantine`. See the rule state section.

???+ note

    The rule name sent to HarfangLab is the Sigma `title`, prefixed with `alert_prefix` when set. The API accepts at most 100 characters, so a longer name is truncated and a warning is issued.

### Rule state

HarfangLab does not store `global_state` next to the agent flags, it derives each from the other:

| global_state | enabled | block_on_agent | quarantine_on_agent | Behaviour |
| ------------ | ------- | -------------- | ------------------- | --------- |
| disabled     | false   | false          | false               | The rule is not evaluated |
| alert        | true    | false          | false               | The rule raises an alert |
| block        | true    | true           | false               | The matching activity is blocked on the agent |
| quarantine   | true    | true           | true                | The matching file is quarantined on the agent |

`droid` settles the state before sending it, so only the combinations above are ever pushed. In practice:

- setting `global_state` sets the agent flags accordingly
- setting `block_on_agent` or `quarantine_on_agent` raises the state, so asking for a block on a rule left in the default `alert` state is honoured rather than silently dropped

???+ warning

    The API schema advertises a `backend_alert` value. HarfangLab stores it as `alert`, which would have `droid` update the rule on every run. It is normalised to `alert` with a warning.

### Sigma Custom Fields

| Custom Field | Values     | Description |
| ------------ | ---------- | ----------- |
| disabled     | true/false | Set to true if you want to disable the rule |
| removed      | true/false | Set to true if you want to delete the rule |
| harfanglab   | mapping    | Per rule override of the platform defaults |

The `harfanglab` mapping accepts `global_state`, `hl_status`, `block_on_agent` and `quarantine_on_agent`, which override the values set in the configuration file for this rule only.

```yaml title="example_sigma.yml"
custom:
    #disabled: True
    #removed: True
    harfanglab:
        global_state: block # (1)!
        hl_status: stable
```

1.  This rule is blocked on the agent, while the rest of the rules keep the default state of the configuration file.

???+ note

    The Sigma `level` is sent as the rule level override on the platform, and the Sigma `references` are sent as the rule references. Both are sent on every export, so removing them from the rule clears them on the platform as well.

### Correlation rules

Sigma [correlation rules](https://sigmahq.io/docs/meta/correlations.html) are supported and exported to the source set in `source_id_correlation`.

HarfangLab resolves the rules a correlation depends on from the YAML document itself, so the atomic rules a correlation refers to travel along in the same document. They are marked so that they are not compiled as standalone rules on the platform.

???+ warning

    A referenced rule must be defined in the same file as the correlation rule. A correlation rule cannot refer to a rule already deployed on the platform in another file.

A correlation type the backend cannot express is reported as a warning and the rule is skipped, in the same way as an unsupported field.

???+ note

    The correlation rule overrides `pipelines_correlation` and `format_correlation` described in the [configuration](../configuration.md) page are not needed here, since the backend emits Sigma for both atomic and correlation rules.

### Integrity check

`droid rules integrity` retrieves the rule from the platform and compares:

- the Sigma content, parsed rather than compared as raw text, since HarfangLab re-serialises the document it stores
- the rule name
- the enabled state, against the `disabled` custom field
- the parsing feedback returned by the Sigma engine of the platform

A rule marked as `removed` is expected to be absent from the platform.
