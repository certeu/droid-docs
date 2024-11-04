# Getting started

## Installation
---

`droid` is published as a [Python package] and can be installed with `pip` in a virtual environment or in a Docker container. The installation is straightforward:

- Install `droid` and the desired Sigma [backends] and [pipelines]
- [Configure](./configuration.md) your droid configuration file
- Configure your Sigma [pipelines](./configuration.md#configure-your-plaform-pipelines)
- Optional: Validate your rules using pySigma validators

If you already have a repository containing all your Sigma rules or if you wish to start from scratch we made available a [repository](https://github.com/certeu/droid-init) to start with `droid` by cloning it.

### Install droid

1. Open up a terminal and install `droid` with:

=== "Latest"

    ```sh
    pip install detect-droid
    ```

=== "0.1.X"

    ```sh
    pip install detect-droid=="0.1.X"
    ```

[Python package]: https://pypi.org/project/detect-droid/

### Install the Sigma [backends]

[backends]: https://sigmahq.io/docs/digging-deeper/backends.html
[pipelines]: https://sigmahq.io/docs/digging-deeper/pipelines.html
[product]: https://sigmahq.io/docs/basics/rules.html#logsources
[category]: https://sigmahq.io/docs/basics/rules.html#logsources
[output formats]: https://sigmahq.io/docs/digging-deeper/backends.html#output-formats

For instance, if you intend to use the following backends:

```bash
pip install pysigma-backend-splunk
pip install pysigma-backend-kusto
pip install pySigma-backend-elasticsearch
```

???+ note

    You will find the full list of backends on the Sigma documentation [page](https://sigmahq.io/docs/digging-deeper/backends.html#available).

### Install additional Sigma pipelines

This is optional but you can install additional pipelines. For instance:

```bash
pip install pysigma-pipeline-windows # (1)!
```

1. Windows logsource to Channel field and generic logsource to Windows audit events mapping

???+ note

    You will find the full list of pipelines on the Sigma documentation [page](https://sigmahq.io/docs/digging-deeper/backends.html#available). Select "Pipelines" from "Plugin Type".
