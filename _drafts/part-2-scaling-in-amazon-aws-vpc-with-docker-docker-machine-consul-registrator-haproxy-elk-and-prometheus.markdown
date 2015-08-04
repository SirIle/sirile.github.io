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

In the [previous post]({ %post_url 2015-07-28-scaling-with-discovery-on-docker-swarm-with-consul-registrator-and-haproxy-with-prometheus-monitoring-and-elk-log-aggregation %}) I demonstrated scaling on a local VirtualBox environment using HAProxy based load balancing with added service discovery and scaling over multiple nodes on Docker Swarm. In this article I show how the scripts slightly modified can be used to control an environment running on [Amazon Virtual Private Cloud (VPC)](https://aws.amazon.com/vpc/) environment.

## Setting up the Amazon VPC
