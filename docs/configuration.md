# Configure droid

The tool is flexible and requires a [TOML](https://toml.io/en/) (.toml) configuration file for various tasks. If you have cloned the [example repository](https://github.com/certeu/droid-init), just open `droid_config.toml`.

The next sections will highlight the config per sections.

### Configure the basics

```toml
[base]

raw_rules_directory = "rules/raw" # (1)!

sigma_validation_config = "validation/validate.yml" # (2)!
...
```

1.  This is the location of the raw rules directory.

2.  This is the location of the Sigma validation config. See the appropriate section below.

### Configure the platform

Next, configure your platform in the `platforms` section. You will find in the [platforms](./platforms.md) page the required parameters.

```toml
[platforms.supported_platform]

...
```

???+ warning

    Make sure your platform is supported [here](./platforms.md).

### Configure your plaform pipelines

`droid` ease the conversion of the Sigma rules for the desired platform by leveraging the Sigma pipelines. It applies the platforms you desire **by log sources**.

`droid` will look for a match with the Sigma `logsource` fields and the platform pipelines config.

```yaml title="proc_access_win_svchost_credential_dumping.yml" hl_lines="12 13 14"
title: Credential Dumping Attempt Via Svchost
id: 174afcfa-6e40-4ae9-af64-496546389294
status: test
description: Detects when a process tries to access the memory of svchost to potentially dump credentials.
references:
    - Internal Research
author: Florent Labouyrie
date: 2021/04/30
modified: 2022/10/09
tags:
    - attack.t1548
logsource:
    product: windows
    category: process_access
detection:
    selection:
        TargetImage|endswith: '\svchost.exe'
        GrantedAccess: '0x143a'
    filter_main_known_processes:
        SourceImage|endswith:
            - '\services.exe'
            - '\msiexec.exe'
    condition: selection and not 1 of filter_main_*
falsepositives:
    - Unknown
level: high
```

```toml title="droid_config.toml" hl_lines="4 5"
[platforms.splunk.pipelines.windows_process_access]

pipelines = ["pipelines/splunk_process_access.yml", "splunk_windows"]
product = "windows"
category = "process_access"
```

In the above example, when the Sigma logsource `product` and `category` of the Sigma rules will match the configuration `platforms.splunk.pipelines.windows_process_access`, the Sigma pipelines will be loaded:

- `splunk_windows` is part of the pySigma Splunk [backend](https://github.com/SigmaHQ/pySigma-backend-splunk)
- `pipelines/splunk_process_access.yml` is a custom pipeline to apply the necessary transforms

???+ info

    Make sure to install the appropriate pySigma backend for the supported platform. See the [installation](./getting-started.md) section.

```yaml title="splunk_process_access.yml" hl_lines="14 15 16 17"
name: Splunk Windows process access
priority: 100

# Author: Mathieu LE CLEACH
# Purpose if this pipeline:
# Process the windows/process_access rules in Splunk

transformations:
  - id: index_condition
    type: add_condition
    conditions:
      index: windows_sysmon
      splunk_server: "*prod.planet-express.local"
    rule_conditions:
      - type: logsource
        category: process_access
        product: windows

postprocessing:
- type: template
  template: |+
    {{ query }} | table _time,host,user,SourceImage,TargetImage,CallTrace,GrantedAccess
```

???+ info

    If droid does not find a match, it will process the Sigma rule as "non-configured" since the log sources was not configured for the given platform. For the purpose of the automation through CI/CD pipelines, droid will not issue an error exit-code for non-configured log sources.

### Configure the validation

You can validate the Sigma rules on the syntax level by leveraging the pySigma [rule validation](https://sigmahq-pysigma.readthedocs.io/en/latest/Rule_Validation.html). It requires a validation configuration file that can be placed in your repository.

```yaml  title="validate.yml"
validators:
    - all
    - -tlpv1_tag
    - -escaped_wildcard
```

???+ note

    Make sure to provide the path to your [droid configuration](#configure-the-basics).