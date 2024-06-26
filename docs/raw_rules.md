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

???+ info

    When using the `raw rules` the features to validate and convert the rules will be possible. Currently, `droid` does not support any custom validation rule for the raw rules. However, when using `droid` in a CI/CD pipeline workflow, you can run another script of your own to validate these rules.

