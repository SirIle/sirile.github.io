---
layout: post
title: "Logstash and Logspout on Docker"
---

## General

When I was using fat containers, I used rsyslog to pass the log messages to a centralized
container which had Elasticsearch, Logstash and Kibana installed. This combination is also
known as the ELK-stack. After moving towards microcontainers I wanted to separate the logging
logic from the application containers. Orchestration is still an issue with using multiple
simple containers, but at least the application containers shouldn't need to know anything
about the logging architecture as long as they can log to stdout and stderr so that Docker
can pick the logs up.

## Separating containers

I decided to install Logstash and Elasticsearch to the same container for simplicity and
separate Kibana as an own container as it's not needed for the logging to work. I use
[Logspout](https://github.com/progrium/logspout) to push the logs from the Docker socket
to the Logbox.

## Creating the Dockerfiles

The Dockerfiles for both Logstash + Elasticsearch (logbox) container and the Kibana (kibanabox) 
container can be found from [github](https://github.com/SirIle/microboxes). They are pretty
straightforward, although they are based on BusyBox to minimize the size. Still the Logbox
is over 480MB in size.

## Running the logging architecture

Orchestration is an area that is still maturing in Dockerspace. The Microcontainer architecture
requires the use of volume containers for sharing the Java 8 runtime and at the moment not
too many orchestrators support that. The containers can be started with the following bash 
script:

{% highlight bash %}
#!/bin/bash
docker run -d --name logbox -h logbox -v /etc/localtime:/etc/localtime:ro -v /etc/timezone:/etc/timezone:ro -p 5000:5000/udp -p 9200:9200 --volumes-from java8filebox --volumes-from basefilebox sirile/logbox
docker run -d -p 8000:80 -h kibanabox -v /etc/localtime:/etc/localtime:ro -v /etc/timezone:/etc/timezone:ro --name kibanabox sirile/kibanabox
docker run -d --name logspout -h logspout -v /etc/localtime:/etc/localtime:ro -v /etc/timezone:/etc/timezone:ro -v /var/run/docker.sock:/tmp/docker.sock progrium/logspout syslog://10.10.10.30:5000
{% endhighlight %}

This expects the host used to run Docker to be 10.10.10.30. Of course a hostname could be used instead.

The Kibana UI can be found from http://10.10.10.30:8000.

## Troubleshooting

At first after having everything running I still couldn't get Logspout messages to go through to
the Logstash container. I couldn't find any error messages and if I used telnet to directly
connect the Logstash container in port 5000, my messages showed up nicely.

After battling for a couple of hours I realized that *UDP* isn't automatically supported
when exposing ports to the host. Instead it needs to be manually specified with adding
"/udp" after the -p parameter.

After adding the parameter the logging system began to work as expected.

## Conclusion

Separating logging using Logspout to direct the logs to Logstash and Elasticsearch works pretty well.
A major problem is that almost nothing seems to log in a compatible syslog format, so a lot
of custom parsing needs to be added to the Logstash side.
