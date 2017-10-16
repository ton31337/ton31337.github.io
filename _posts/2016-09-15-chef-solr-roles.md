---
layout: post
title: Solr search for Chef sucks
categories:
- blog
---

If you use custom attributes like `default['something']['role']` or `default['something']['roles']` you gonna have some problems entirely. Let me show you an example.

Here we have `roles` attribute which says we have `prometheus` role applied on the certain node.
```
chef (12.9.41)> node['roles']
 => ["prometheus"]
```

Let's add some custom attribute with `roles` keyword:
```
chef (12.9.41)> node['xxx-prometheus']['roles']
 => {"imm-leaf"=>9100}
chef (12.9.41)>
```

You can make sure you don't mixed something. This will tell you, what roles node has and with what precedence level:
```
chef (12.9.41)> node.debug_value('roles')
 => [["set_unless_enabled?", false], ["default", :not_present], ["env_default", :not_present], ["role_default", :not_present], ["force_default", :not_present], ["normal", :not_present], ["override", :not_present], ["role_override", :not_present], ["env_override", :not_present], ["force_override", :not_present], ["automatic", ["imm-machine", "machine", "imm-prometheus", "prometheus", "grafana"]]]
```

There you go:
```
% knife search 'roles:imm-leaf' -i
1 items found

us-imm-prometheus1.xxx.io
```
or using log files:
```
% curl "localhost:8983/solr/select?q=content:role__=__imm-leaf" | grep numFound
% tail -f /var/log/opscode/opscode-solr4/current
2017-10-09_12:32:06.62128 18061355949 [qtp1725097945-17] INFO  org.apache.solr.core.SolrCore  ? [collection1] webapp=/solr path=/select params={q=content:roles__%3D__imm-leaf&indent=off&start=0&fq=%2BX_CHEF_database_CHEF_X:chef_40be7d3ebda916803709240aad351d14+%2BX_CHEF_type_CHEF_X:node&sort=X_CHEF_id_CHEF_X+asc&rows=1000&wt=json} hits=18 status=0 QTime=2
```

When you are searching for a node index, `roles` is the key for Solr and roles matches. Solr doesn't always work as expected, thus avoid using `roles` keyword in your custom attributes.
