---
layout: post
title: "Part 2: Scaling in Amazon AWS VPC with Docker, Docker Machine, Consul, Registrator, HAProxy, ELK and Prometheus"
date: "2015-08-04 14:33"
---

{::options parse_block_html="true" /}
<div class="toc">
Contents

* toc
{:toc}
</div>

## General

In the [previous post]({ %post_url 2015-07-28-scaling-with-discovery-on-docker-swarm-with-consul-registrator-and-haproxy-with-prometheus-monitoring-and-elk-log-aggregation %}) I demonstrated scaling on a local VirtualBox environment using HAProxy based load balancing with added service discovery and scaling over multiple nodes on Docker Swarm. In this article I show how the updated scripts can be used to control an environment running on [Amazon Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/) environment.

## Setting up the Amazon VPC

1) Create the VPC instance and note down the VPC id
2) Create a subnet for that VPC, I used IP range 10.10.0.0/16
3) Create an Internet Gateway
4) Create a route between 0.0.0.0/0 and the created Internet Gateway under Route Tables->Routes

After you have used Docker Machine to create the first node the security group Docker+machine should be created. I had to add a rule "ALL TRAFFIC" within the security group (so that the target for the rule is the security group's own id) to get visibility between the Swarm nodes.

## Setting up awscli

[AWS Command Line Interface](http://aws.amazon.com/cli/) is a convenient way to configure settings in the VPC.

## Setting up environment variables for docker

The scripts check the existence of environment variable __AWS_ACCESS_KEY_ID__.

{% highlight bash %}
export AWS_ACCESS_KEY_ID=<secret key>
export AWS_SECRET_ACCESS_KEY=<secret access key>
export AWS_VPC_ID=<vpc-id>
export AWS_DEFAULT_REGION=<region>
export AWS_ZONE=<zone>
{% endhighlight %}

## Setting up the environment

To set up an environment with the infra node, three swarm instances and ten instances of the test service the following commands can be used

{% highlight bash %}
git clone https://github.com/SirIle/docker-multihost.git && cd docker-multihost/swarm
./createInfraNode.sh
for i in {0..2}; do ./createSwarmNode.sh $i; done
for i in {1..10}; do ./startService.sh hello/v1 sirile/node-test; done
{% endhighlight %}

Then you need to 

## Caveats

As the communication between nodes happens inside the VPC and all the Consuls have been configured to advertise the VPC internal IP addresses joining the local Consul to the Consul network isn't feasible unless there is a VPN tunnel between the local host and the AWS VPC.
