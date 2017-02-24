---
layout: post
title: The cost of infrastructure automation
categories:
- blog
---

This year we have started using automation for everything as much as we could including: network automation, server provisioning, application deployments.

Every part has its own limitations and requirements. For network automation we use [Ansible](https://www.ansible.com/) because it has flexible template mechanism, many 3rd party modules, it's fast and really fits our needs. Servers are provisioned with Ansible too but configurations are pushed using our first-class citizen [Chef](https://www.chef.io/){:target="_blank"}{:rel="nofollow"} and some parts are handled with [Consul](https://www.consul.io/){:target="_blank"}{:rel="nofollow"} + [consul-template](https://github.com/hashicorp/consul-template){:target="_blank"}{:rel="nofollow"}.

Why not run a single tool for all things?
Because every tool has its own limitations:

* Ansible doesn't have a mechanism to run itself periodically and it's not as much flexible as Chef;
* Chef doesn't have orchestration layer;
* Consul.. OK, but who would be responsible to bootstrap Consul cluster and other dependencies around it?

We make changes on Github by creating a pull request and [Jenkins](https://jenkins.io/) pulls these changes. Before Github introduced [Code better with Reviews](https://github.com/blog/category/ship), we used just :+1:,:-1: as comment to approve or reject changes. Thanks to this new feature, we are allowed to do it more transparently. Our most maintained Chef repository has enabled `Require pull request reviews before merging` protection, which means code review is mandatory. After pulling changes from Github, Jenkins starts build and does the rest. Some builds have multiple Jenkins slaves for executing specific builds, like building [LXC](https://linuxcontainers.org/){:target="_blank"}{:rel="nofollow"} containers or [Docker](https://www.docker.com/){:target="_blank"}{:rel="nofollow"} containers, which requires different Linux distributions. As explained above we exploit Jenkins for every kind of automation. In most cases we have two or three different builds per Github repository:

* ansible-network - apply changes in production from master branch;
* ansible-network-pr - bootstrap development environment and apply changes from pull request;
* ansible-repo - check the syntax for every playbook.

Automation is the way we work at Hostinger. We don't SSH into the server and do not do any changes. We do changes locally on personal laptop first, later push the code to [Github](https://github.com/) as Pull Request. SSH to server is necessary only for ad-hoc debugging, everything else is dynamically adjusted by Chef, thus there is no point to log in. Most of our user facing servers are identical (depends on role), thus we just pick single one to verify configuration quickly.

Before any new feature is deployed into our current stack we have to think about automation FIRST. We have two environments: development and production. At first, changes go to development environment, where we have more or less identical infrastructure (virtual) and changes are seen quickly after merge. When everything is fine with the development environment, we are free to use the same versions of cookbooks in production environment. Just up another pull request with increased versions.

When we provision a new server, it is automatically detected by our monitoring platform [Prometheus](https://prometheus.io/), clusters are reconfigured according to decent cluster size and so on, nothing else has to be done manually. If someone breaks or changes something in configuration files, everything is reverted back automatically by `chef-client` which runs every 7 minutes in background. Some services need to react faster than every 7 minutes, thus consul-template helps here. We have consul cluster per region and consul-template is running as client where it's needed. As an example we use consul-template for regenerating upstreams for [Openresty](http://openresty.org/){:target="_blank"}{:rel="nofollow"}. It requires nearly real-time operation.

Network automation is done using primary tool Ansible. We use [Cumulus](http://cumulusnetworks.com/) network operating system which allows us to have a fully automated network where we reconfigure the network including BGP neighbors, firewall rules, ports, bridges, etc. on changes. Nothing is changed directly inside the switch. Cumulus has a virtual instance called Cumulus VX which allows us to converge all changes before pushing them to production. Jenkins build converges Cumulus VX locally, applies Ansible playbook, does tests. If everything is fine then we are happy too. For instance, we add a new node, then Ansible will automatically see changes in Chef inventory by looking at LLDP attributes and regenerate network configuration for particular switch. If we want to add a new BGP upstream or firewall rule, we just create a pull request to our Github repo and everything is done automatically including checking syntax and deploying changes in production. You can find more information about our network stack in the previous blog [post](http://www.hostinger.com/blog/engineering/awex-ipv6){:target="_blank"}{:rel="nofollow"}.

### Other small yet nice to have automation

We internally use [Slack](https://slack.com/) and it's common sense to do work as much as possible in the chat. We have automated Jenkins builds, Github hooks, for example: new issue or pull request is created, then notification to Slack channel is sent. Or we can start Jenkins build directly from channel by typing: `ada j b 22`, put the website to sleeping state `ada sleep <url>` and so forth.

### Takeaways

* You can automate 80% of tasks, it's not necessary to cover 100%;
* You have more time to spend on more interesting tasks instead of copy pasting the same around fleet of servers;
* Knowledge sharing: all organization members are able to see, comment, do changes freely;
* No secrets between team mates, infrastructure as a code is visible by everyone;
* Server which is not under automation costs more time and money than automated.
