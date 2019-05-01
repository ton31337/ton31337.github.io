---
layout: post
title: Git in operations
categories:
- blog
---

I was asked from developers to tell about git how we use it in operations team (aka SRE/DevOps). In an ideal world, developers should teach operation guys how to use git at scale. But sometimes reality is far away.

Below are my short slides I prepared one hour before stand-up, keep it simple, stupid.

<div style="left: 0; width: 100%; height: 0; position: relative; padding-bottom: 74.9288%;"><iframe src="//speakerdeck.com/player/4a0b12d46226415e9a4afb00b2f16add" style="border: 0; top: 0; left: 0; width: 100%; height: 100%; position: absolute;" allowfullscreen scrolling="no" allow="autoplay; encrypted-media"></iframe></div>

I will explain each command from the slides how I use it not only for work but for open-source contributions as well.

### git config --list

List all the configurations per project or globally. I don't think I use it at all, just for the setup phase maybe.

### git config --global push.default current

This is very handy to avoid specifying the branch name you want to push. This allows you to use the current branch you are working right now by default.

### git config --global branch.master.remote origin

Usually, you have to type for instance `git push origin master`. Using this configuration knob and the previous one, you cut this command to just `git push`.

The trick for Github users `git config --global remote.origin.fetch +refs/pull/*/head:refs/remotes/pr/*`. This command fetches the branches by pull request numbers. If a pull request has a number `123`. Then you can check out locally to that branch by typing `git checkout pr/123`. Cool? Not? Continue reading.

### git config --global alias.last 'log -1 HEAD'

I always tend to use `git log -n 1`. Using git aliases it could be more human readable like `git last`.

### git remote add upstream https://

When you work on a local fork of an upstream repository where you don't have write permissions you are out of sync of branches between your fork and the upstream. `git fetch --all` helps you to have all branches an upstream has locally as well.

### git log

Mostly I use `git log`, but if I want to find a relevant commit id by commit message I use `git log --oneline | grep nginx` or `git log --grep 'nginx'`. If I need to find the commit by looking into the code, not only to commit message I use `git log -S 'nginx'`.

### git shortlog

I rarely use this command. Mostly I use it when I want to see what are the most active contributors to the arbitrary project or it's useful to have this information when publishing release notes.

The output is well known for some people:

```
Donatas (2241):
      Merge pull request #1 from hostinger/feature/add_supermarket_cookbook
      Merge pull request #2 from hostinger/feature/add_hostinger-machine_cookbook
```

`git shortlog -s -n --all --no-merges` gives us more statistic related view:

```
% git shortlog -s -n --all --no-merges
  2685  Donatas Abraitis
   415  FirstName LastName
```

### git add

Every day I usually use `git add .` to add all files recursively in a current directory, but sometimes I have to use `git add <path>` to add specific file or directory to skip some temporary files. I never used `git add --interactive`. It's quite fun but doesn't look very handy at a first glance. It just slowdowns the process, IMHO.

I use `git add --patch` not often, but it's useful if you changed more than one code places and finally, you need to add only a few of them (not all). Patch splits your changes into hunks and you are able to select which one to include.

To remove the file you accidentally added you could do that by typing `git rm --cached <path>`.

### git commit

Sorry, all IDE _lovers_, but `git commit -m` is really big crap. First of all, it teaches you to not think much about the commit message and description at all. Why? Because if you use `-m` you probably want to push this commit as fast as possible without caring about the details. Lazy developers syndrome:

```
* git add .
* git commit -m 'I do not care much about that'
* git add .
* git commit -m 'I do not care much about that 2'
* git add .
* git commit -m 'fix'
* git push
```

Let's talk about `git commit --signoff`. Some big projects require you to sign before merging.

_It is used to say that you certify that you have created the patch in question, or that you certify that to the best of your knowledge, it was created under an appropriate open-source license, or that it has been provided to you by someone else under those terms._

### git commit --amend

I always use `git commit --amend --allow-empty --signoff` when committing. Amend is the feature which allows you to modify previous commit keeping the same commit message, description. It doesn't create an additional commit, just appends the code you want. It's required to have a single commit per pull request somewhere because some linters or CI/CD platforms run tests under commit and not under pull request. What happens when you have two commits where the later one fixes syntax while the formerly introduced syntax error. It would start two deployments per commit and both should fail.

### bad commit example

```
commit a5140910088f33ec6edd3869a1354ebfafb63ff8
Author: joni2back <xxx@gmail.com>
Date:   Tue Mar 3 16:58:54 2015 -0300

    fix error
```

What can you extract from this message? Absolutely absurd. Nothing. Oh, I can say that the author is doing his career well.

### good commit example

```
commit afad5cedf1be827238b376e63b0b93bb555c928e
Author: Donatas Abraitis <donatas.abraitis@gmail.com>
Date:   Mon Feb 25 21:16:02 2019 +0200

    bgpd: Add peer action for PEER_FLAG_IFPEER_V6ONLY flag

    peer_flag_modify() will always return BGP_ERR_INVALID_FLAG because
    the action was not defined for PEER_FLAG_IFPEER_V6ONLY flag.

    ```
    global PEER_FLAG_IFPEER_V6ONLY = 16384;
    global BGP_ERR_INVALID_FLAG = -2;

    probe process("/usr/lib/frr/bgpd").statement("peer_flag_modify@/root/frr/bgpd/bgpd.c:3975")
    {
        if ($flag == PEER_FLAG_IFPEER_V6ONLY && $action->type == 0)
                printf("action not found for the flag PEER_FLAG_IFPEER_V6ONLY\n");
    }

    probe process("/usr/lib/frr/bgpd").function("peer_flag_modify").return
    {
        if ($return == BGP_ERR_INVALID_FLAG)
                printf("return BGP_ERR_INVALID_FLAG\n");
    }
    ```
    produces:
    action not found for the flag PEER_FLAG_IFPEER_V6ONLY
    return BGP_ERR_INVALID_FLAG

    $ vtysh -c 'conf t' -c 'router bgp 20' -c 'neighbor eth1 interface v6only remote-as external'

    Signed-off-by: Donatas Abraitis <donatas.abraitis@gmail.com>
```

A bit better example of how good commit message should look like. I don't say it's ideal, but it's good enough to understand what's going on.

### git rebase

Most of the time I use `git rebase --interactive master`. But if an arbitrary project has release branches I use for instance `git rebase --interactive stable/7.0`.

### git stash

I don't use stash often, but it's a cool feature git provides. It's like a buffer or storage where you put your changes without committing and can jump between branches. Let's say I work on a feature branch and I need to switch to the master branch to pull new changes. Not possible if I have some changes already. I need to stash my changes to the buffer and retrieve those changes later using `git stash pop` or restore back to more accurate revision. You can list all stashes with `git stash list`.

### git reset

Let's say I committed a bad commit or amended to the previous commit without creating a commit before (overwritten someone's changes). In such cases I use `git log -n 2` to grab the commit id I want to reset to. `git commit <sha1>`. Then I'm able to commit my changes again.

Git allows you to use the `--hard` or `--soft` method for reset action. With soft you lose only the commit history. With hard, you lose the changes as well.

### git tag

I'm not a fan of using tags. But some projects use them. Basically, there are two types of tags: annotated and lightweight. The latter one just marks the commit with arbitrary tag while the former one creates some metadata in the history, like data, author, message.

### git bisect

I discovered this feature a few months ago. It's very cool. Want to find the commit which broke functionality or which caused regression? Bisect can help you. You specify a good and a bad commits before the start:

```
git bisect start
git bisect bad <sha1>
git bisect good <sha1>
git bisect bad
...
git bisect reset
```

Or you can automate this by using `git bisec run <cmd>`. Where _cmd_ is the script which handles bad/good commits by exit code.

### git cherry-pick

What if I want to pick the commit which is already on the master and _recommit_ it to another branch, let's say `stable/7.0`? Create one more commit or pick the same commit which is on the master branch and backport it to the `stable/7.0`? `git cherry-pick <sha1>` does the trick.

### git reflog

I used `reflog` I think fewer times than I have fingers on my hands. But it's a really tough feature. For instance, if I used to amend and broke my commit, I'm able to recover to the previous state of the same commit by _traveling in time_ ;-) It's like a git time machine.

### Conclusion

I don't realize my daily work without git. It's a critical tool for day to day work even in operations. Truly stupid, but even lawyers nowadays are  using Github to publish chapters publicly.

When you code for work, you mostly do not need git command such as git bisect or git cherry pick and so on. That's why I'm always looking for open-source contributions - to learn something new again and again.

Keep commits as small as possible to help to bisect truly relevant commit instead of reviewing elephant one.
