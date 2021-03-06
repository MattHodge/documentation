---
title: Prometheus Check
kind: documentation
---

## Overview

This page first looks at the `PrometheusCheck` interface, and then proposes a simple Prometheus Check that collects timing metrics and status events from [Kube DNS](https://github.com/DataDog/integrations-core/blob/master/kube_dns/datadog_checks/kube_dns/kube_dns.py).

## Prometheus check interface 

`PrometheusCheck` is [a mother class](https://github.com/DataDog/dd-agent/blob/master/checks/prometheus_check.py) providing a structure and some helpers to collect metrics, events, and service checks exposed via Prometheus. Minimal configuration for checks based on this class include:


- Overriding `self.NAMESPACE`
- Overriding `self.metrics_mapper`
- Implementing the `check()` method
AND/OR
- Create method named after the prometheus metric they will handle (see `self.prometheus_metric_name`)

## Your first Prometheus check
Let's write a simple kube_dns check:

### Configuration

<div class="alert alert-warning">
The names of the configuration and check files must match. If your check
is called <code>mycheck.py</code> your configuration file <em>must</em> be
named <code>mycheck.yaml</code>.
</div>

Configuration for a Prometheus Check is almost the same as a regular Agent Check, please refer to the [dedicated Agent check documentation to learn more](/agent/agent_checks/#configuration)

The main difference is to include the variable `prometheus_endpoint` in your `check.yaml` file. This goes into `conf.d/kube_dns.yaml`:

```yaml
init_config:

instances:
    # url of the metrics endpoint of prometheus
  - prometheus_endpoint: http://localhost:10055/metrics
```

### Writing your Prometheus check
All Prometheus checks inherit from the `PrometheusCheck` class found in `checks/prometheus_check.py`: 

```python
from checks.prometheus_check import PrometheusCheck

class KubeDNSCheck(PrometheusCheck):
```

#### Overriding `self.NAMESPACE`

`NAMESPACE` is the prefix metrics will have. It needs to be hardcoded in your check class:

```python
from checks.prometheus_check import PrometheusCheck

class KubeDNSCheck(PrometheusCheck):
    def __init__(self, name, init_config, agentConfig, instances=None):
        super(KubeDNSCheck, self).__init__(name, init_config, agentConfig, instances)
        self.NAMESPACE = 'kubedns'
```

#### Overriding `self.metrics_mapper`

`metrics_mapper` is a dictionary where the key is the metric to capture and the value is the corresponding metric name in Datadog.
The reason for the override it so metrics reported by the Prometheus checks are not counted as [custom metric](/getting_started/custom_metrics):

```python
from checks.prometheus_check import PrometheusCheck

class KubeDNSCheck(PrometheusCheck):
    def __init__(self, name, init_config, agentConfig, instances=None):
        super(KubeDNSCheck, self).__init__(name, init_config, agentConfig, instances)
        self.NAMESPACE = 'kubedns'
        self.metrics_mapper = {
            #metrics have been renamed to kubedns in kubernetes 1.6.0
            'kubedns_kubedns_dns_response_size_bytes': 'response_size.bytes',
            'kubedns_kubedns_dns_request_duration_seconds': 'request_duration.seconds',
            'kubedns_kubedns_dns_request_count_total': 'request_count',
            'kubedns_kubedns_dns_error_count_total': 'error_count',
            'kubedns_kubedns_dns_cachemiss_count_total': 'cachemiss_count'
        }
```

#### Implementing the check method

From `instance` we just need `endpoint` which is the metrics endpoint to use to poll metrics from Prometheus:

```python
def check(self, instance):
    endpoint = instance.get('prometheus_endpoint')
```

##### Exceptions

If a check cannot run because of improper configuration, programming error, or
because it could not collect any metrics, it should raise a meaningful exception. This exception is logged and is shown in the Agent [info command](/agent/faq/agent-status-and-information) for easy debugging. For example:

    $ sudo /etc/init.d/datadog-agent info

      Checks
      ======

        my_custom_check
        ---------------
          - instance #0 [ERROR]: Unable to find prometheus_endpoint in config file.
          - Collected 0 metrics & 0 events

Improve your `check()` method with `CheckException`:

```python
from checks import CheckException

def check(self, instance):
    endpoint = instance.get('prometheus_endpoint')
    if endpoint is None:
    raise CheckException("Unable to find prometheus_endpoint in config file.")
```

Then as soon as you have data available you just need to flush:

```python
from checks import CheckException

def check(self, instance):
    endpoint = instance.get('prometheus_endpoint')
    if endpoint is None:
    raise CheckException("Unable to find prometheus_endpoint in config file.")
    # By default we send the buckets.
    if send_buckets is not None and str(send_buckets).lower() == 'false':
        send_buckets = False
    else:
        send_buckets = True

    self.process(endpoint, send_histograms_buckets=send_buckets, instance=instance)
```

### Putting It All Together

```python
from checks import CheckException
from checks.prometheus_check import PrometheusCheck

class KubeDNSCheck(PrometheusCheck):
    """
    Collect kube-dns metrics from Prometheus
    """
    def __init__(self, name, init_config, agentConfig, instances=None):
        super(KubeDNSCheck, self).__init__(name, init_config, agentConfig, instances)
        self.NAMESPACE = 'kubedns'

        self.metrics_mapper = {
            # metrics have been renamed to kubedns in kubernetes 1.6.0
            'kubedns_kubedns_dns_response_size_bytes': 'response_size.bytes',
            'kubedns_kubedns_dns_request_duration_seconds': 'request_duration.seconds',
            'kubedns_kubedns_dns_request_count_total': 'request_count',
            'kubedns_kubedns_dns_error_count_total': 'error_count',
            'kubedns_kubedns_dns_cachemiss_count_total': 'cachemiss_count',
        }

    def check(self, instance):
        endpoint = instance.get('prometheus_endpoint')
        if endpoint is None:
            raise CheckException("Unable to find prometheus_endpoint in config file.")

        send_buckets = instance.get('send_histograms_buckets', True)
        # By default we send the buckets.
        if send_buckets is not None and str(send_buckets).lower() == 'false':
            send_buckets = False
        else:
            send_buckets = True

        self.process(endpoint, send_histograms_buckets=send_buckets, instance=instance)
```


## Getting further

You can improve your Prometheus check with the following methods: 

### `self.ignore_metrics`

Some metrics are ignored because they are duplicates or introduce a very high cardinality. Metrics included in this list will be silently skipped without a `Unable to handle metric` debug line in the logs

### `self.labels_mapper`

If the `labels_mapper` dictionary is provided, the metrics labels names in the `labels_mapper` will use the corresponding value as tag name when sending the gauges.

### `self.exclude_labels`

`exclude_labels` is an array of labels names to exclude. Those labels will just not be added as tags when submitting the metric.

### `self.type_overrides`

`type_overrides` is a dictionary where the keys are Prometheus metric names and the values are a metric type (name as string) to use instead of the one listed in the payload. It can be used to force a type on untyped metrics.
Note: it is empty in the mother class but will need to be overloaded/hardcoded in the final check not to be counted as custom metric.