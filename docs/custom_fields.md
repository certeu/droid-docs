# Rule custom fields

You can place custom fields in your rules to:

- Disable a rule: `disabled: True`
- Remove a rule: `removed: True`

Additionally:

- `ignore_search` to `True` will skip the search step
- `ignore_export_error` to `True` will ignore the export error and integrity error

???+ info

    The `ignore_export_error` option currently only works with raw detection rules.

Some of the settings placed in your `droid` configuration file can be replaced as well.

```yaml title="example_sigma.yml" hl_lines="25-36"
title: Load Undocumented Autoelevated COM Interface
id: fb3722e4-1a06-46b6-b772-253e2e7db933
status: test
description: COM interface (EditionUpgradeManager) that is not used by standard executables.
references:
    - https://www.snip2code.com/Snippet/4397378/UAC-bypass-using-EditionUpgradeManager-C/
    - https://gist.github.com/hfiref0x/de9c83966623236f5ebf8d9ae2407611
author: oscd.community, Dmitry Uchakin
date: 2020/10/07
modified: 2021/11/27
tags:
    - attack.defense_evasion
    - attack.privilege_escalation
    - attack.t1548.002
logsource:
    category: process_access
    product: windows
detection:
    selection:
        TargetImage|contains: 'EXCEL.EXE'
    condition: selection
falsepositives:
    - Unknown
level: high
custom:
    #ignore_search: True
    disabled: True
    #removed: True
    #ignore_export_error: True
    earliest_time: -1h@h
    latest_time: now
    cron_schedule: '5 * * * *'
    alert.suppress: "0"
    #alert.suppress.fields: "customer,host,user"
    alert.digest_mode: "1"
    actions: "webhook"
    action.webhook.param.url: "https://automation.pizza-planet.local/34432/44232"
```

You can also replace the webhook URL at the rule level by using `action.webhook.param.url: "$WEBHOOK_URL_SOMETHING"`.