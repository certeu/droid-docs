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

As of today, the basic authentication is supported. If API Tokens are prefered, feel free to contribute.

- `basic`: Authenticate using username and password

When using the Azure Registation App, load the following environment variables:

- `DROID_ELASTIC_USERNAME` : Elastic username
- `DROID_ELASTIC_PASSWORD` : Elastic password

### Supported Platforms

For Sigma rules both `esql` (ES|QL) and `eql` (EQL) are supported. However since only esql supports correlations rules, it is advised to use esql only. There are no known benefits to using EQL over ES|QL.

For raw rules, both esql (ES|QL) and eql (EQL) are supported. Always use the `raw_language` custom field to specify the language used in the Sigma yaml file.

For non correlation ES|QL rules, the metadata fields are automatically added to the query for better deduplication by the Elastic backend. For correlation ES|QL rules this is not needed since correlation rules are already deduplicated.

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


