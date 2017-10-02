---
layout:           post
title:            "Send AWS Elastic Beanstalk logs from Docker to AWS Elasticsearch 5.3"
date:             2017-10-01 20:15:00 +0200
last_modified_at: 2017-10-01 20:15:00 +0200
tags:             [aws, docker, gelf, log, elastic beanstalk, elasticsearch]
introduction:     "How to send your application logs, access logs and system logs from your AWS Elastic Beanstalk Multicontainer Docker Environment to AWS Elasticsearch Service 5.3."
---

A couple of months ago when I sat up my multicontainer docker environment on AWS Elastic Beanstalk I realized that I could only export my logs manually from the Elastic Beanstalk user interface. So I started looking into ways of sending my logs in real-time from my docker environment to some external service where I could archive and search them easily at a later point for statistics and debugging purposes.

After some googling, I found that docker has a gelf (Graylog Extended Log Format) log driver, so I ended up choosing AWS Elasticsearch as my logging service since it is easy to query in Elasticsearch and also to create graphs and visual output of your data in Kibana. I tried to just simply add a new section in my Dockerrun.aws.json file that changed the log driver of the docker container to gelf [*[reference]*](http://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_LogConfiguration.html).

However, this did not work as expected since the AWS ECS agent does not have the gelf log driver activated by default, it has to be activated before the agent starts. The only way I found to achieve this is by adding gelf log driver support to the AWS ECS agent through a custom Elastic Beanstalk extension and then restarting the AWS ECS process.

I set up my environment by adding the following two Elastic Beanstalk extensions to the root of my GitHub repository. The contents of the first file `.ebextensions/gelf.config` goes as follows:
{% highlight plaintext %}
files:
  "/home/ec2-user/setup-available-log-drivers.sh":
    mode: "000755"
    owner: root
    group: root
    content: |
      #!/bin/sh
      set -e
      if ! grep gelf /etc/ecs/ecs.config &> /dev/null
      then
        echo 'ECS_AVAILABLE_LOGGING_DRIVERS=["json-file","syslog","gelf"]' >> /etc/ecs/ecs.config
      fi

container_commands:
  00-configure-gelf:
    command: /home/ec2-user/setup-available-log-drivers.sh
{% endhighlight %}

The contents of the second ebextension file `.ebextensions/restart-docker.config` should be:
{% highlight plaintext %}
commands:
  01stopdocker:
    command: "sudo stop ecs > /dev/null 2>&1 || /bin/true && sudo service docker stop"
  02killallnetworkbindings:
    command: "sudo killall docker > /dev/null 2>&1 || /bin/true"
  03removenetworkinterface:
    command: "rm -f /var/lib/docker/network/files/local-kv.db"
    test: test -f /var/lib/docker/network/files/local-kv.db
  09restart:
    command: "service docker start && sudo start ecs && sleep 120s"
{% endhighlight %}

Once those two files are in place, gelf logging support should be activated in the AWS ECS agent on the next deploy, auto scale or environment rebuild on Elastic Beanstalk. Now it should be as easy as just adding a logstash container to the Dockerrun.aws.json file to run a logstash container in the multicontainer docker environment.

{% highlight json %}
{
  "AWSEBDockerrunVersion": 2,
  "volumes": [
    {
      "name": "app",
      "host": {
        "sourcePath": "/var/app/current"
      }
    }
  ],
  "containerDefinitions": [
    {
      "name": "logstash",
      "image": "oliverlundquist/logstash:5.5.2",
      "essential": true,
      "memory": 1024,
      "mountPoints": [
        {
          "sourceVolume": "app",
          "containerPath": "/var/app/current"
        }
      ],
      "portMappings": [
        {
          "hostPort": 12201,
          "containerPort": 12201,
          "protocol": "udp"
        }
      ]
    },
    {
      "name": "nginx",
      "image": "oliverlundquist/nginx:latest",
      "essential": true,
      "memory": 512,
      "mountPoints": [
        {
          "sourceVolume": "app",
          "containerPath": "/var/app/current"
        }
      ],
      "portMappings": [
        {
          "hostPort": 80,
          "containerPort": 80
        }
      ],
      "links": [
        "php"
      ],
      "logConfiguration": {
        "logDriver": "gelf",
        "options": {
          "gelf-address": "udp://localhost:12201"
        }
      }
    }
  ]
}
{% endhighlight %}

You can use any logstash docker image that you would like, however, if you would like to use the one that I have build listed in the Dockerrun.aws.json file above, you need to add two logstash configuration files in a .logstash folder in the root of your repository.

One file `.logstash/logstash.conf`, which corresponds to the `--path.config` setting in logstash which is your pipeline configuration file and another file `.logstash/logstash.yml` which is the logstash settings configuration file which is set with the `--path.settings` argument.

I hope that this helps someone out there that is looking into ways of how to send logs to AWS Elasticsearch from a multicontainer docker environment in AWS Elastic Beanstalk.
