---
hide:
  - navigation
---

# droid documentation

`droid` is a PySigma wrapper allowing an easy adoption of [Sigma](https://sigmahq.io/) and helps enabling Detection-As-Code. The ultimate goal of `droid` is to consume a repository Sigma rules and deploy them on one or multiple platform (SIEM/EDR).

The tool also supports native SIEM/EDR search queries.

![droid workflow](./resources/droid_workflow.png)

<div class="grid cards" markdown>

-   :material-clock-fast:{ .lg .middle } __Set up in few minutes__

    ---

    Install [`detect-droid`](/droid-docs/getting-started/#installation) with [`pip`](https://pypi.org/project/detect-droid/) and get up
    and running in minutes

    [:octicons-arrow-right-24: Getting started](./getting-started.md)

-   :fontawesome-solid-gears:{ .lg .middle } __Enable automation__

    ---

    Enable detection content versioning and take advantage of Sigma at scale

    [:octicons-arrow-right-24: Sigma Unleashed: A Realistic Implementation](https://www.first.org/resources/papers/conf2024/1315-1350-Sigma-Unleashed-Mathieu-Le-Cleach.pdf)

-   :simple-toml:{ .lg .middle } __Made to measure__

    ---

    Standardise the detection rules on your platforms, change the rules settings and more with few lines

    [:octicons-arrow-right-24: Configuration](/droid-docs/configuration/)

-   :material-scale-balance:{ .lg .middle } __Open Source, EUPL__

    ---

    droid is licensed under the EUPL and available on [GitHub](https://github.com/certeu/droid)

    [:octicons-arrow-right-24: License](https://github.com/certeu/droid/blob/main/LICENSE)

</div>

## Features

Key features are:

1. **Validate** the syntax of Sigma rules
2. **Convert** them by applying a set of transforms per log source and platform
3. **Search** in logs and report on findings
4. **Test** the rules by leveraging Atomic Red Team™ (work in progress)
5. **Deploy** them with any compatible SIEM and EDR

[droid]: https://github.com/certeu/droid

## Supported SIEM/EDR


???+ info

    See the list of supported platforms (SIEM/EDR) [here](./platforms/index.md).