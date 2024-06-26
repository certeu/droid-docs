---
hide:
  - navigation
---

# droid documentation

`droid` is a PySigma wrapper allowing an easy adoption of Sigma and helps enabling Detection-As-Code. The ultimate goal of `droid` is to consume a repository Sigma rules and deploy them on one or multiple platform (SIEM/EDR).

The tool also supports native SIEM/EDR search queries.

![droid workflow](./resources/droid_workflow.png)

---

**FIRST presentation slides:** coming soon.

**Source Code:** [https://github.com/certeu/droid](https://github.com/certeu/droid)

---

## Features

Key features are:

1. **Validate** the syntax of Sigma rules
2. **Convert** them by applying a set of transforms per log source and platform
3. **Search** in logs and report on findings
4. **Test** the rules by leveraging Atomic Red Team™ (work in progress)
5. **Deploy** them with any compatible SIEM and EDR (.e.g. Splunk, Microsoft Sentinel).

[droid]: https://github.com/certeu/droid

## Supported SIEM/EDR


???+ info

    The tool currently supports only Splunk and Microsoft Sentinel for the search and export feature. Additional platform support will be added in future updates.

???+ note

    It is also possible to use a plain SIEM/EDR [search query](/raw_rules/) instead of Sigma.