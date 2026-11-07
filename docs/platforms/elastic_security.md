- [x] Search feature
- [x] Export feature
    * [x] Remove detection rules
    * [x] Disable detection rules
- [ ] Detection rule actions

### Index detection

???+ warning

    It is necessary to specify the indexes within the Sigma pipeline or it will be defaulted to `logs-*`.

=== "Single index"

    ```yaml
    name: Elastic Windows Process Creation
    priority: 100

    transformations:

    - id: index_set_state
      type: set_state
      key: "index"
      val: "windows-sysmon.*"
      rule_conditions:
        - type: logsource
          product: windows
          category: process_creation
    ```

=== "Multiple index"

    ```yaml
    name: Elastic Windows Process Creation
    priority: 100

    transformations:

    - id: index_set_state
      type: set_state
      key: "index"
      val:
        - "windows-sysmon.*"
        - "windows-sysmon2.*"
      rule_conditions:
        - type: logsource
          product: windows
          category: process_creation
    ```

### Authentication

The same credentials are used for Kibana (export and integrity) and for Elasticsearch (search). Set `auth_method` to one of:

- `basic`: Authenticate using username and password, loaded from the environment variables:
    - `DROID_ELASTIC_USERNAME` : Elastic username
    - `DROID_ELASTIC_PASSWORD` : Elastic password
- `api_key`: Authenticate using an Elasticsearch API key, loaded from the environment variable:
    - `DROID_ELASTIC_API_KEY` : the **encoded** API key, as returned in the `encoded` field when the key is created

```toml
[platforms.elastic]
auth_method = "api_key"
```

### Supported Platforms

For Sigma rules both `esql` (ES|QL) and `eql` (EQL) are supported. However since only esql supports correlations rules, it is advised to use esql only. There are no known benefits to using EQL over ES|QL.

For raw rules, both esql (ES|QL) and eql (EQL) are supported. Always use the `raw_language` custom field to specify the language used in the Sigma yaml file.

For non correlation ES|QL rules, the metadata fields are automatically added to the query for better deduplication by the Elastic backend. For correlation ES|QL rules this is not needed since correlation rules are already deduplicated.

### Query format

Converted ES|QL queries read like the ones written in Kibana: the source command first, then one upper-case command per line.

```
FROM logs-endpoint.events.process-* METADATA _id, _index, _version
| WHERE process.name=="powershell.exe" AND `1vendor.process.args` LIKE "*-enc*"
```

ES|QL and EQL only accept a field name without backticks when every segment of it starts with a letter (or with `_` or `@`) and is not a reserved word. Droid quotes every other field name for you, which covers:

- a segment starting with a digit, e.g. a vendor prefix like `1vendor.category`
- a segment that is a reserved word, e.g. `delivery.by` or `rule.first` in ES|QL, `rule.sequence` in EQL
- any character outside letters, digits and `_`, e.g. `process.parent-name`

???+ note

    Upgrading to a version of droid that changes the query format updates every deployed rule on the next export, and reports an integrity mismatch for each rule until it is exported again.

### MITRE ATT&CK

The `attack.*` tags of a rule fill its MITRE ATT&CK mapping in Kibana:

- a technique is listed under every tactic it belongs to, e.g. `attack.t1078` under Initial Access, Persistence, Privilege Escalation and Stealth
- a subtechnique is nested under its technique, e.g. `attack.t1078.004`
- a tactic tag without technique, e.g. `attack.initial-access` or `attack.ta0001`, keeps its own entry

The other tags are exported as Kibana tags.

### Severity and risk score

| Sigma level   | Severity | Risk score |
| ------------- | -------- | ---------- |
| critical      | critical | 99         |
| high          | high     | 73         |
| medium        | medium   | 47         |
| low           | low      | 21         |
| informational | low      | 1          |

Elastic has no informational severity, the risk score keeps those rules apart from the low ones.

### Main config

| Parameter              | Mandatory | Default Value | Description                                                                                                 |
| ---------------------- | --------- | ------------- | ----------------------------------------------------------------------------------------------------------- |
| kibana_url             | Yes       | N/A           | The Base URL to your Kibana, since the Security API used goes via Kibana not Elasticsearch                  |
| auth_method            | No        | "basic"       | `basic` or `api_key`, see [Authentication](#authentication)                                                 |
| elastic_hosts          | No        | []            | Elasticsearch hosts, only used when Searching                                                               |
| kibana_ca              | No        | False         | Certificate Chain used on the Kibana host                                                                   |
| elastic_ca             | No        | None          | Certificate Chain used on the Elastic host, can also be the same as Kibana, only used when Searching        |
| elastic_tls_verify     | No        | False         | Used to turn on and off Certificate Validation for Elastic Connections, only used when Searching            |
| schedule_interval      | No        | "1"           | Interval at which the alert rule should run                                                                 |
| schedule_interval_unit | No        | "h"           | Interval unit second (s), minute (m), hour (h) or day (d)                                                   |
| additional_look_back   | No        | None          | Extra time each run looks back, e.g. "1m", so events ingested late are not missed between two runs          |
| license                | No        | "DRL"         | The license of your rule                                                                                    |
| legacy_esql            | No        | false         | This flag is only if you are on an Elastic Version that needs the square brackets around the metadata info. |
| building_block_prefix  | No        | "BB"          | Optional prefix for Building Block Rules to distinguish from Alert rules                                    |
| alert_prefix           | No        | None          | Optional prefix to your imported rules to distinguish them                                                  |
| esql_search_range_gte  | No        | "now-1h"      | How far back an ES\|QL search looks, unless the rule sets its own search look-back                          |
| eql_search_range_gte   | No        | "now-24h"     | How far back an EQL search looks, unless the rule sets its own search look-back                             |


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
additional_look_back = "1m" # Each run covers the last 6 minutes
license = "DRL"
alert_prefix = "SIGMA"
legacy_esql = false

# Pipeline example
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


