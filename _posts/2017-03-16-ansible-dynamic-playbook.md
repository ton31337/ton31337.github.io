---
layout: post
title: Incremental deployment with Ansible
categories:
- blog
---

Previous month I [wrote](http://blog.donatas.net/blog/2017/02/02/ansible-converge-time/) how we reduced converge time to 1h for the whole infrastructure. Today I can say, it's way too much to wait for changes. We wanted to see changes faster than waiting an hour or sometimes unpredictable amount of time.

Day by day run list is getting fatter, thus the challenge was to keep this task as simple as possible and do not introduce any complex layer underneath. We made a resolution (raising eyebrow) to involve simple versioning mechanism for roles.

The idea is very simple:

* Keep current versions in Redis;
* Iterate over roles and compare current role's version with the new one;
* If they differ, use this role for this build.

This improved build just deploys roles their versions were increased. To have fully consistent state between all nodes we trigger another Jenkins's build like `ansible-hostinger-full` to reflect all changes across all nodes equally. But with this incremental deployment, we can see changes rapidly.

Here is an example how to manipulate dynamic playbook file:

```
class AnsibleDeploy
  def initialize(source, destination)
    @source = source
    @destination = destination
    @run_list = {}
    @current_roles = {}
    @redis = Redis.new
  end

  def run_list
    Dir['roles/*/version'].each do |file|
      name = role_name(file)
      version = role_version(file)
      @current_roles[name] = get_role_version(name)
      update_role_version(name, version)
    end
    render_playbook
  end

  private

  def role_name(file)
    file.match(%r{roles\/(.*)\/version})[1]
  end

  def role_version(file)
    File.read(file).chop
  end

  def get_role_version(role)
    @redis.hmget("ansible:role:#{role}", 'version').first
  end

  def update_role_version(role, version)
    @redis.hmset("ansible:role:#{role}", 'version', version)
    @run_list[role] = version
  end

  def diff
    @run_list.map do |role, version|
      next if @current_roles[role] == version
      role
    end.compact
  end

  def roles_from_yaml
    YAML.load_file(@source).last['roles']
  end

  def deploy_info_from_yaml
    YAML.load_file(@source).map do |playbook|
      playbook.delete('roles')
      playbook
    end
  end

  def render_playbook
    playbooks = deploy_info_from_yaml.map do |deploy|
      if mapping.any?
        deploy.merge('roles' => mapping)
      else
        deploy
      end
    end
    File.write(@destination, playbooks.to_yaml)
  end

  def mapping
    roles_from_yaml.map do |role|
      if role.is_a? String
        role if diff.include?(role)
      else
        role if diff.include?(role['role'])
      end
    end.compact
  end
end

AnsibleDeploy.new('hostinger.yml', 'hostinger-dynamic.yml').run_list
```

Build looks like:

```
% ruby ./ansible_deploy.rb
% ansible-playbook hostinger-dynamic.yml -i hosts
```

Everything is trivial, take `hostinger.yml` file as a source, strip `roles` part from this `YAML` and generate the new one `hostinger-dynamic.yml` with appropriate `run_list`.

#### Conclusion

* You have inconsistent state if automation fails;
* Make simple tasks simple;
* Ansible sucks in either way :)
