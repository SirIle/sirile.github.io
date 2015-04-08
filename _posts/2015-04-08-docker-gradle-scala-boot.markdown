---
layout: post
title:  "Spring Boot with Scala on Docker using Gradle"
---

## General

I started experimenting with building containers using Gradle as part of a development cycle with minimum duration feedback cycle and maximum portability for Microservices. I'll write a bit more about that later, but thought that documenting this process of creating and publishing a container would be useful.

At the same time I wanted to combine Spring Boot and Scala, so this also acts as an example for that.

### TL;DR

The example can be run in local VirtualBox Docker environment with:

{% highlight bash %}
docker-machine create -d virtualbox dev && $(docker-machine env dev)
git clone https://github.com/SirIle/scala-boot-test.git && cd scala-boot-test
./gradlew container
docker run -p 80:80 --rm sirile/scala-boot-test
{% endhighlight %}

And just as easily the container could be located in Amazon Web Services or Google Compute Engine or Azure.

## Prerequisites

In this example I'm using OS X but all of the components should be as easily installable and runnable on Windows. I use [brew](http://brew.sh/) to install the packages, but you can also install all of the mentioned tools by following the instructions on their homepage.

### Docker Machine

Using [Docker Machine](https://github.com/docker/machine) provided environment for building the containers makes things very simple. With Docker Machine you can use both a local development environment and cloud based solutions as easily. In this article I will use the VirtualBox based local environment for building the container and running the example.

### VirtualBox

The local Docker Machine environment uses [VirtualBox](https://www.virtualbox.org/) to host the boot2docker instance.

### Docker client

[Gradle-docker](https://github.com/Transmode/gradle-docker) plugin uses the docker client to communicate with the Docker Machine instance.

### Installing the prerequisites

{% highlight bash %}
brew install docker
brew cask install docker-machine
brew cask install virtualbox
{% endhighlight %}

## Creating the local Docker Machine environment

### Creating the instance

{% highlight bash %}
docker-machine create -d virtualbox dev
{% endhighlight %}

### Pointing local Docker client to dev instance

{% highlight bash %}
$(docker-machine env dev)
{% endhighlight %}

### Finding out the IP of the created dev instance

The IP of the created instance can be checked with

{% highlight bash %}
$ docker-machine ls
NAME    ACTIVE   DRIVER       STATE     URL                         SWARM
dev     *        virtualbox   Running   tcp://192.168.99.100:2376
{% endhighlight %}

or with `docker-machine ip dev`.

## Checking out the project

The [repository](https://github.com/SirIle/scala-boot-test) can be checked out with

{% highlight bash %}
git clone https://github.com/SirIle/scala-boot-test.git
{% endhighlight %}

## Running the example locally

{% highlight bash %}
./gradlew run
{% endhighlight %}

### Testing the example

Now you should be able to point the browser to http://localhost:8080/hello.

## Building the container

Container is built by running

{% highlight bash %}
./gradlew container
{% endhighlight %}

This uses the local docker client which has been configured to use the Docker Machine environment for building the container. The first time build will take some time as the base image containing the JRE is downloaded from the docker.io.

After the build is finished, the created image can be checked with

{% highlight bash %}
$ docker images
REPOSITORY               TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
sirile/scala-boot-test   latest              962a1edb8789        17 seconds ago      199.2 MB
sirile/minijavabox       latest              35cbd95c692b        8 days ago          178.7 MB
{% endhighlight %}

## Running the container

Container can be run with

{% highlight bash %}
docker run -p 80:80 --rm sirile/scala-boot-test
{% endhighlight %}

## Testing the container

The application should be available in the address that you can query with `docker-machine ip`. For example http://192.168.99.100 and in the port 80 that was exposed when running the container.
