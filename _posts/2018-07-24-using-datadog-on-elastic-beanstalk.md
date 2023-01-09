---
layout: post
title: Using DataDog on AWS Elastic Beanstalk
tags: aws datadog logs elastic-beanstalk
---

There is somewhat limited information available on how to use DataDog on the Elastic Beanstalk platform, and it took me quite a bit of time to get it working. Here is a summary of how to make the DataDog agent run and report logs to DataDog.

First, place two files in predefined locations: a minimal DataDog configuration file with an API key, DataDog endpoint, and a key enabling log collection, and a second file containing a list of log files you would like to watch and send to DataDog.

```yaml
files:
  "/etc/datadog-agent/datadog.yaml":
    mode: "000644"
    owner: root
    group: root
    content: |
      dd_url: https://app.datadoghq.com
      api_key: 012301230123012301230123
      logs_enabled: true
  "/etc/datadog-agent/conf.d/app.d/app.yaml":
    mode: "000644"
    owner: root
    group: root
    content: |
      logs:
       - type: file
         path: /var/log/application.json.log
         service: application
         source: php
```

Next, execute the command to install the DataDog agent, noting that the install script returns an error code if you try to reinstall the agent. Therefore, you should include a guard checking for the existence of the /opt/datadog-agent directory before executing the install command:

```yaml
commands:
  install_datadog:
    command: "[ -d /opt/datadog-agent ] || curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh | sudo DD_API_KEY=012301230123012301230123 bash"
```

Optionally, you may also choose to restart the agent on every application deployment. This is not strictly necessary, but provided here for completeness:

```yaml
files:
  ...
  "/opt/elasticbeanstalk/hooks/appdeploy/post/50_restart_services":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/usr/bin/env bash
      set -xe
      . /opt/elasticbeanstalk/support/envvars
      ...
      if status datadog-agent | grep -q start ; then
        echo "Restarting DataDog"
        restart datadog-agent
      else
        echo "Starting DataDog"
        start datadog-agent
      fi
      ...
```

And that's it!