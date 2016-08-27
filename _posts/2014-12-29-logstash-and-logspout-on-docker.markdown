---
layout: post
title: "Logstash and Logspout on Docker"
---

## Update 26th June 2015

**A newer version with more recent versions of the components can be found
[here]({% post_url
2015-06-26-elasticsearch-logstash-kibana-and-logspout-on-docker %}).**

The old examples may not run properly as the images have been updated with new
versions of tools. Especially kibanabox needs to be built by hand as the
DockerHub now builds the Kibana 4.1 image.

## General

When I was using fat containers, I used rsyslog to pass the log messages to a
centralized container which had Elasticsearch, Logstash and Kibana installed.
This combination is also known as the ELK-stack. After moving towards
microcontainers I wanted to separate the logging logic from the application
containers. Orchestration is still an issue with using multiple simple
containers, but at least the application containers shouldn't need to know
anything about the logging architecture as long as they can log to stdout and
stderr so that Docker can pick the logs up.

All the containers in the examples have the corresponding docker.io builds
created for them, so running the scripts is enough without the need to build the
containers locally.

## Separating containers

I decided to install Logstash and Elasticsearch to the same container for
simplicity and separate Kibana as an own container as it's not needed for the
logging to work. I use [Logspout](https://github.com/progrium/logspout) to push
the logs from the Docker socket to the Logbox.

## Creating the Dockerfiles

The Dockerfiles for both Logstash + Elasticsearch (logbox) container and the
Kibana (kibanabox) container can be found from
[github](https://github.com/SirIle/microboxes). They are pretty straightforward,
although they are based on BusyBox to minimize the size. The Logbox image is
about 180MB in size.

### Minicontainer image

I also created a minibox image [(as opposed to microbox)]({% post_url
2014-12-31-fat-containers-microcontainers-and-minicontainers %}) which contains
the Java 8 runtime so it can be ran as standalone without the need to include
the volume container for the JRE and runit at runtime. The size is 351.1MB and
the Dockerfile can be found from
[github](https://github.com/SirIle/miniboxes/tree/master/minilogbox).

## Running the logging architecture

Orchestration is an area that is still maturing in Dockerspace. As not many
orchestrators support volume containers, the minicontainer version can be more
feasible.

### Running with microcontainers

The Microcontainer architecture requires the use of volume containers for
sharing the Java 8 runtime . The filebox volume containers need to be created
before running the example and can be created with:

~~~bash
#!/bin/bash
docker run --name=basefilebox sirile/basefilebox
docker run --name=java8filebox sirile/java8filebox
~~~

The containers can be started with the following bash script for
microcontainers:

~~~bash
#!/bin/bash
docker run -d --name logbox -h logbox -p 5000:5000/udp -p 9200:9200 --volumes-from java8filebox --volumes-from basefilebox sirile/logbox
docker run -d -p 8000:80 -h kibanabox --name kibanabox sirile/kibanabox
docker run -d --name logspout -h logspout -v /var/run/docker.sock:/tmp/docker.sock progrium/logspout syslog://10.10.10.30:5000
~~~

### Running with minicontainers

The minicontainer image includes the Java 8 JRE and runit so including the
volume containers isn't necessary. This simplifies portability.

~~~bash
#!/bin/bash
docker run -d --name logbox -h logbox -p 5000:5000/udp -p 9200:9200 sirile/minilogbox
docker run -d -p 8000:80 -h kibanabox --name kibanabox sirile/kibanabox
docker run -d --name logspout -h logspout -v /var/run/docker.sock:/tmp/docker.sock progrium/logspout syslog://10.10.10.30:5000
~~~

### Notes

This example expects the host used to run Docker to be 10.10.10.30. Of course a
hostname could be used instead if the container can resolve the name using the
defined DNS.

The Kibana UI can be found from [http://10.10.10.30:8000](http://10.10.10.30:8000).

## Troubleshooting and Issues

### UDP

At first after having everything running I still couldn't get Logspout messages
to go through to the Logstash container. I couldn't find any error messages and
if I used telnet to directly connect the Logstash container in port 5000, my
messages showed up nicely.

After battling for a couple of hours I realized that *UDP* isn't automatically
supported when exposing ports to the host. Instead it needs to be manually
specified with adding "/udp" after the -p parameter.

After adding the parameter the logging system began to work as expected.

### Syslog message parsing

Another issue I faced were a lot of parsing failures for the log messages. It
was caused by Logspout using ISO 8601 as the date format whereas the older
rsyslog based parsing expected syslog format. After modifying the parser the
_grokparsefailures stopped and the messages were properly handled.

### Timestamps and timezones

For the timestamps to be the same across the host and containers, it may be
advisable to pass `-v /etc/localtime:/etc/localtime:ro -v
/etc/timezone:/etc/timezone:ro` as parameters for all the container run
commands.

## Conclusion

Separating logging using Logspout to direct the logs to Logstash and
Elasticsearch works pretty well. One issue is that a lot of the programs log in
a non-syslog friendly way by for example indenting the log messages. Parsing a
Java stacktrace from multiple messages can be painful and requires further
investigation. With rsyslog the messages can be combined before sending, but
logspout forwards the messages as they appear, so that is not an option.
