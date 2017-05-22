---
layout: post
title: From few hours to a reasonable convergence time
categories:
- blog
---

Last week we tackled a huge upgrade process where we moved our current Hostinger infrastructure to be fully automated. What was before, I will leave to your imagination.

Hence, after few days of discussions about what tools to use we decided to continue with [Ansible](https://www.ansible.com/). Why Ansible? Because it was the simplest solution for the short amount of time and few engineers on the way. Almost everyone had at least some experience with it, thus move on. This took only a few sprints, which was a deadline. A goal without a deadline is merely a dream, thus we did it.

In short, we converted our old infrastructure (custom compiled PHP, Nginx, Apache, etc.) to a more sustainable approach ([CloudLinux](https://www.cloudlinux.com/), CageFS, PHP selector, Openresty and so on and so forth). Since the upgrade we have been able to do pro-active monitoring which we didn't have before, it allows us to spot the biggest problems sooner.

Early days were hard. Ansible was better than horrible, but it's really not flexible as Chef, which we already use more for internal infrastructure and [000webhost.com](https://www.000webhost.com/) project. We were stuck on almost every second task and didn't know how to best workaround them, but it's good enough for now.

As we wrote in previous posts we use Jenkins for all sort of deployments. This was also not an exception. Once we tested Ansible convergence in a development environment, everything was smooth and fast, deploys took 3-5 min. on average. Unfortunately, in the production deployments took hours, which was a huge pain.

# Performance optimizations for Ansible

#### Pipelining + ControlPersist

This was the first thing to enable. After it was turned on, we achieved 2x improvements in convergence time. Great success for a while, we reduced the time from more than 3h (3 hr 28 min) to ~2h (1 hr 48 min) per build. This was good as a first optimization, but there was still room left for improvements. We could reduce this time by running a single Ansible play, but we have a knob as a pre-flight check to deploy changes to development environment first. If it succeeds, then continue with the production environment.

```
[ssh_connection]
ssh_args = -C -o ControlMaster=auto -o ControlPersist=600s
control_path = ~/.ssh/ansible-ssh-%%h-%%p-%%r
pipelining = True

```

#### Make sure to minimize 'changed' tasks

Before making any improvements I suggest to turn on profiling for Ansible tasks:

```
[defaults]
callback_whitelist = profile_tasks
```

This will let you see most critical paths like:

```
10:56:39 setup ------------------------------------------------------------------ 28.48s
10:56:39 hostinger-cloudlinux : Upgrade alt-python-pip -------------------------- 12.96s
10:56:39 hostinger-cagefs : Remove some default commands from CageFS ------------- 3.60s
10:56:39 hostinger-init : Remove unnecessary packages ---------------------------- 2.49s
```

Avoid using `state=latest` if you do not keep your system up to date, this will return almost immediately instead of looking for newest versions.

```
- name: Install Apache and dependencies
  yum:
    name: "{{ item }}"
    state: latest
  with_items:
    - httpd
    ...

```

If you are using `yum` module with external sources like:

```
- name: Enable remi-repo
  yum:
    name: http://rpms.famillecollet.com/enterprise/remi-release-{{ ansible_distribution_major_version }}.rpm
  when: remi.stat.exists == false
```

Make sure to check if repository file exists or not, because yum downloads this file every time:

```
- name: Check if remi-repo is installed
  stat:
    path: /etc/yum.repos.d/remi.repo
  register: remi
```

Another trick we did, we disabled `update` for git module so that it does not check out the latest version every time, added `creates` option for almost every `unarchive`, `uri`, `shell` modules and so on.

#### Use linting for playbooks

And of course to add some sugar always use [ansible-lint](https://github.com/willthames/ansible-lint). It simplifies development process to keep infrastructure code clean, valid and understandable for everyone. For instance, before linting our playbooks returned such an output:

```
% ansible-lint *.yml | grep ANSIBLE | sort -rn | uniq -c
  14 ANSIBLE0016 Tasks that run when changed should likely be handlers
  46 ANSIBLE0013 Use shell only when shell functionality is required
  22 ANSIBLE0012 Commands should not change things if nothing needs doing
   6 ANSIBLE0011 All tasks should be named
   2 ANSIBLE0010 Package installs should not use latest
   4 ANSIBLE0009 Octal file permissions must contain leading zero
   2 ANSIBLE0006 chkconfig used in place of service module
  10 ANSIBLE0004 Git checkouts must contain explicit version
   6 ANSIBLE0002 Trailing whitespace
```

Now we run `ansible-lint` for every pull request. If it fails - we need to fix it before going to merge.

After these changes, our build time dropped to 1 hr 0 min.

![Ansible deployment](/images/ansible_jenkins_deploy.png)

# Wrapping up

* Always use profiling tools if performance matters;
* Agentless automation tools are slower than agent-based in its nature.
