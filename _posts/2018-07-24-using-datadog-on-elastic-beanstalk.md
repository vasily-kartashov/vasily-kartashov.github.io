---
layout: post
title: Using DataDog on AWS Elastic Beanstalk
---

There are a somewhat limited infromation on how to use DataDog on Elastic Beanstalk platform and it took me personally quite a bit of time to get it working. So here's a short summary of how to make the datadog agent run and also report logs to datadog.

So the first thing that is need to be done is placing a couple of files in predefined locations:

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

The first file is a minimal datadog configuration file with an API key, DataDog endpoint and a key enabling log collection. The second file contains a list of log files we'd like to watch and send to DataDog.

Next, we need to execute command installing datadog agent

```yaml
commands:
  install_datadog:
    command: "[ -d /opt/datadog-agent ] || curl -L https://raw.githubusercontent.com/DataDog/datadog-agent/master/cmd/agent/install_script.sh | sudo DD_API_KEY=012301230123012301230123 bash"
```

Here's to note that install script returns error code if you try to reinstall the agent. That's why we need the guard in front of it checking the existence of the `/opt/datadog-agent` directory.

Next we would like to restart the agent on every application deployment. Stricktly speaking it's not necessary and you might skip this step. I'll provide it here for completeness:

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

And that would be it.