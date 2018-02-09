---
title: Environment
kind: documentation
---

#### Definition

An environment is a first class dimension that you can use to scope a whole Datadog APM application. Some display settings can be shared across environments, but all the measurable data (traces/metrics/statistics) can not be re-aggregated across multiple environments. Use cases can be:

* Stage environments such as production, staging, and pre-production
* Datacenters and availability zones in isolation

Environments are [tags](/agent/tagging), therefore they must follow the following rules:

* They must start with a lower case letter.
* Other characters must be alphanumeric lower case Unicode characters, underscores, minuses, colons, periods or slashes.
* They must not be more than 100 characters long.

Environments in traces and configuration files are normalized:

* Unsupported characters are replaced by underscores.
* Upper case characters are converted to lower case.

#### Default environment

The default environment for un-tagged data is `env:none`. See below to see how to specify custom environments.:

**Note**: if you are using environments, you still get a default `env:none` environment where all the non-tagged data goes.

#### Setup

There are several ways to specify an environment when reporting data:

1. Host tag:

    If you use a host tag that looks like `env:XXXX`, all traces reported from that agent are tagged accordingly.

2. Agent configuration:

    Override the default tag used by the trace agent in [the Agent configuration file](/agent/faq/where-is-the-configuration-file-for-the-agent). This tags all traces coming through the agent, overriding the host tag value.

    ```
    [trace.config]
    env = pre-prod
    ```

3. Per trace:

   When submitting a single trace, specify an environment by tagging one of its spans with the metadata key `env`. This overrides the agent configuration and the host tags values (if any).

    ```
    # in code this looks like
    span.set_tag('env', 'prod')
    ```