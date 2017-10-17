---
layout: post
title: Incremental deployment with Ansible, part 2
categories:
- blog
---

In previous [post](http://blog.donatas.net/blog/2017/03/16/ansible-dynamic-playbook/) I described how [we](https://www.hostinger.com/) use Ansible in production environments for much faster deployments than coping with defaults. In this blog post, I will explain another one (simplified version) way to deal with incremental changes.

I borrowed the idea from [git-stacktrace](https://github.com/pinterest/git-stacktrace). The idea is just to looks for the changed files with the latest commit and generate playbook file dynamically.

But there is one drawback comparing with versioning. It's not possible(?) to run partial playbook if `group_vars` and/or `host_vars` were modified unless you have a proper mapping between variables and role's name.

```
class AnsibleDeploy
  def initialize(source, destination)
    @source = source
    @destination = destination
    @roles_changed = roles_changed
  end

  def run!
    render_playbook
  end

  private

  def roles_changed
    g = Git.open('.')
    commits = g.log(2)
    g.diff(commits.first.sha, commits.last.sha).stats[:files].map do |file, _|
      parsed = file.match(%r{roles\/([\d\w\-\_]+)\/})
      parsed[1] if parsed
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
        deploy.merge('roles' => roles_from_yaml)
      end
    end
    File.write(@destination, playbooks.to_yaml)
  end

  def mapping
    roles_from_yaml.map do |role|
      if role.is_a? String
        role if @roles_changed.include?(role)
      else
        role if @roles_changed.include?(role['role'])
      end
    end.compact
  end
end

source = ARGV[0]
destination = ARGV[1]

AnsibleDeploy.new(source, destination).run!

```
