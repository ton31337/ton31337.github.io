---
layout: post
title: Service discovery by BGP community
categories:
- blog
---

We use [Consul](https://www.consul.io/) + [consul-template](https://github.com/hashicorp/consul-template) for service discovery and to generate configuration files on changes. With Consul everything is fine, except that you need an external service to register nodes/services/whatever. Hence, overall we have three services needed to implement service discovery. It's getting even harder if you want to have high availability for this service.

Others use for instance Chef/Puppet/Ansible inventory for discovery, but this is too much slow comparing with nearly real-time operations.

I tried to think how can we do simpler without having Consul as a backing store for data, consul-template as a rendering engine and any external service for registrations to Consul. The first idea comes to BGP communities. Why not reuse BGP communities as a service, for instance `12345:<port>`. Services announcing /32 (IPv4) or /128 (IPv6) self as a node which could handle `<port>` for service X. Another endpoint listens for incoming BGP updates and parses BGP communities. It will see, that e.g. prefix `2001::123/128` has `12345:443` community and treat it as it handles connections for port 443.

![exa-template](/images/exa-template.png)

#### Generating template

This is the configuration for the ExaBGP process:

```
neighbor 2001:802::1 {
  router-id 10.10.10.1;
  local-address 2001:802::2;
  local-as 65501;
  peer-as 65501;
  capability {
    route-refresh;
  }
  hold-time 3;
  process exa-template {
    run /usr/local/rvm/rubies/ruby-2.3.0/bin/ruby /etc/exabgp/exa-template.rb;
    receive {
      parsed;
      updates;
      neighbor-changes;
    }
    encoder json;
  }
}
```

The critical option is `route-refresh` capability. This is needed to resend all routes from neighbor periodically if changed. Why it's needed? Because ExaBGP doesn't send all update messages in one group, thus it's not possible to track which attribute or neighbor was changed. In this example, I'm using Ruby language to implement this, but to avoid installing Ruby, this can be rewritten to Python (which is the default language for most of the distributions) or some another language. Here is the simple two-liner script which calls [exa-template](https://github.com/ton31337/exa-template) class:

```
require 'exa-template'

ExaTemplate.new('/etc/exa-template/service.cfg.erb',
        '/etc/servicex/service.cfg').parse_events
```

exa-template itself registers `services` hash, which has a hash of arrays of IP addresses per port. For instance:

```
services = {
  '443' => ['2001:802::123/128', '1.1.1.1/32'],
  '8080' => ['1.1.1.1/32']
}
```

Hence, template looks like:

```
<% services.each do |port, ips| %>
service_<%= port %>
  listen 127.0.0.1:<%= port %>
  <%- ips.each do |ip| -%>
  backend <%= ip.split('/')[0] %>:<%= port %>
  <%- end -%>
<% end %>
```

Final result:

```
service_443
  listen 127.0.0.1:443
  backend 2001:802::123:443
  backend 1.1.1.1:443

service_8080
  listen 127.0.0.1:8080
  backend 1.1.1.1:8080
```

Sounds cool? This is only a proof of concept to show how it's possible to exploit BGP communities to tackle service discovery in Layer3. Finally I would say, that it's not as fast as blinking your eyes, but reasonably fast enough.
