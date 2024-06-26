# Raw rules

`droid` allows the usage of a plain SIEM/EDR query instead of Sigma in the `detection` field. This feature was implemented to make the use of a platform's advanced search while keeping Sigma's metadata.

## Configuration

First, decide where these rules will remain. For instance, these rules can remain at the following path: `rules/raw`. If you have multiple SIEM/EDR, you can create a subdirectory to split the rules by platform, for instance: `rules/raw/siem1`.

Finally, set the path in your configuration file. `droid` will use the argument `-r` and check if the path matches

```toml title="droid_config.toml" hl_lines="3"
[base]

raw_rules_directory = "rules/raw"

sigma_validation_config = "validation/validate.yml"
```

???+ note

    In the above example base configuration file, if we're using `droid -cf droid_config.toml -r rules/raw/splunk -p splunk -i` `droid` will match `rules/raw` and will enable the raw rules features.

## Usage

To use a plain SIEM/EDR search query, just replace the Sigma's `detection` field as the following:

```yaml title="detect_web_rdmn.yml" hl_lines="11"
title: Detect Suspicious Web Activities
id: 1b3b4d57-117b-4635-96a7-7d0d2c1e54b9
status: experimental
description: |
  This Sigma rule detects suspicious web activities by matching web logs against a lookup table of suspicious IPs, user agents, and URLs.
author: Apache25
date: 2024/04/05
logsource:
  product: web
  service: apache
detection: |-
  index=web_logs sourcetype=access_combined
  | lookup suspicious_activities.csv suspicious_ip AS clientip OUTPUT suspicious_ip
  | lookup suspicious_activities.csv suspicious_user_agent AS useragent OUTPUT suspicious_user_agent
  | lookup suspicious_activities.csv suspicious_url AS uri_path OUTPUT suspicious_url
  | eval is_suspicious_ip=if(isnotnull(suspicious_ip), 1, 0)
  | eval is_suspicious_user_agent=if(isnotnull(suspicious_user_agent), 1, 0)
  | eval is_suspicious_url=if(isnotnull(suspicious_url), 1, 0)
  | eval total_suspicious_score=is_suspicious_ip + is_suspicious_user_agent + is_suspicious_url
  | where total_suspicious_score > 0
  | eval suspicious_reason=case(
      is_suspicious_ip==1, "Suspicious IP",
      is_suspicious_user_agent==1, "Suspicious User Agent",
      is_suspicious_url==1, "Suspicious URL",
      total_suspicious_score==2, "Multiple Indicators",
      total_suspicious_score==3, "Highly Suspicious"
    )
  | stats count BY clientip, useragent, uri_path, suspicious_reason
  | sort -count
  | rename clientip AS "Client IP", useragent AS "User Agent", uri_path AS "URL Path", suspicious_reason AS "Reason"
  | fields "Client IP", "User Agent", "URL Path", "Reason", count
  | eval action_required=case(
      count > 10, "Immediate Action",
      count > 5, "Review Required",
      count <= 5, "Monitor"
    )
  | sort -action_required
  | table "Client IP", "User Agent", "URL Path", "Reason", count, action_required
  | rename count AS "Suspicious Activity Count"
fields:
  - clientip
  - useragent
  - uri_path
  - suspicious_reason
  - count
  - action_required
falsepositives:
  - Known internal tests
  - False positives due to misconfiguration
level: high
```

???+ info

    When using the `raw rules` the features to validate and convert the rules will be possible. Currently, `droid` does not support any custom validation rule for the raw rules. However, when using `droid` in a CI/CD pipeline workflow, you can run another script of your own to validate these rules.

