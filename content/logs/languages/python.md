---
title: Python log collection
kind: documentation
further_reading:
- link: "/logs/processing"
  tag: "Documentation"
  text: Learn how to process your logs
- link: "/logs/parsing"
  tag: "Documentation"
  text: Learn more about parsing
- link: "/logs/explore"
  tag: "Documentation"
  text: Learn how to explore your logs
- link: /logs/faq/log-collection-troubleshooting-guide
  tag: "FAQ"
  text: Log Collection Troubleshooting Guide
---

<div class="alert alert-info">
Datadog's Logs is currently available via public beta. You can apply for inclusion in the beta via <a href="https://www.datadoghq.com/log-management/">this form</a>.
</div>

## Overview

Use your favorite python logger to log into a file on your host. Then [monitor this file with your Datadog agent](/logs/languages/python/#configure-the-Datadog-agent) to send your logs to Datadog.

## Setup
Python logs are quite complex to handle, mainly because of tracebacks. They are split into multiple lines which make them difficult to associate with the original log event.  
To address this use case we strongly recommend you use a JSON formatter when logging in order to:

* Ensure each stack_trace is properly wrapped into the correct log
* Ensure that all the attributes of a log event are properly extracted (severity, logger name, thread name, etc…)
 
Here are setup examples for the two following logging libraries:

- [JSON-log-formatter](https://pypi.python.org/pypi/JSON-log-formatter/0.1.0)
- [Python-json-logger](https://github.com/madzak/python-json-logger)

### Log into a file

Usage example with [JSON-log-formatter](https://pypi.python.org/pypi/JSON-log-formatter/0.1.0):

```python
import logging

import json_log_formatter

formatter = json_log_formatter.JSONFormatter()

json_handler = logging.FileHandler(filename='/var/log/my-log.json')
json_handler.setFormatter(formatter)

logger = logging.getLogger('my_json')
logger.addHandler(json_handler)
logger.setLevel(logging.INFO)

logger.info('Sign up', extra={'referral_code': '52d6ce'})
```

The log file will contain the following log record (inline).
```json
{
    "message": "Sign up",
    "time": "2015-09-01T06:06:26.524448",
    "referral_code": "52d6ce"
}
```

Usage example with [Python-json-logger](https://github.com/madzak/python-json-logger):

```python
    import logging
    from pythonjsonlogger import jsonlogger

    logger = logging.getLogger()

    logHandler = logging.StreamHandler()
    formatter = jsonlogger.JsonFormatter()
    logHandler.setFormatter(formatter)
    logger.addHandler(logHandler)
```
 
Once the [handler is configured](https://github.com/madzak/python-json-logger#customizing-fields), the log file contains the following log record (inline):

```json
{
    "threadName": "MainThread",
    "name": "root",
    "thread": 140735202359648,
    "created": 1336281068.506248,
    "process": 41937,
    "processName": "MainProcess",
    "relativeCreated": 9.100914001464844,
    "module": "tests",
    "funcName": "testFormatKeys",
    "levelno": 20,
    "msecs": 506.24799728393555,
    "pathname": "tests/tests.py",
    "lineno": 60,
    "asctime": ["12-05-05 22:11:08,506248"],
    "message": "testing logging format",
    "filename": "tests.py",
    "levelname": "INFO",
    "special": "value",
    "run": 12
}
```

### Configure the Datadog agent

Create a file `conf.yaml` in the Agent's `python.d/` directory with the following content:

```yaml
init_config:

instances:

##Log section
logs:

    ## - type : file (mandatory) type of log input source (tcp / udp / file)
    ##   port / path : (mandatory) Set port if type is tcp or udp. Set path if type is file
    ##   service : (mandatory) name of the service owning the log
    ##   source : (mandatory) attribute that defines which integration is sending the logs
    ##   sourcecategory : (optional) Multiple value attribute. Can be used to refine the source attribtue
    ##   tags: (optional) add tags to each logs collected

  - type: file
    path: /path/to/your/python/log.log
    service: myapplication
    source: python
    sourcecategory: sourcecode
```

Then [Restart the Agent](/agent/faq/start-stop-restart-the-datadog-agent) to apply the configuration change.

## Further Reading

{{< partial name="whats-next/whats-next.html" >}}
