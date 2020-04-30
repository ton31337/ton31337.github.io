---
layout: post
title: Two years in FRRouting
categories:
- blog
---

Since the beginning of my contribution to [FRRouting](https://frrouting.org/), I raised myself to the top 15 contributors (that's a huge win for me personally):

```
% git shortlog --summary --numbered | head -n15
  4713    Donald Sharp
  1712    Quentin Young
  1674    David Lamparter
  1108    Renato Westphal
   791    paul
   545    Philippe Guibert
   522    Lou Berger
   518    Russ White
   464    Paul Jakma
   446    Rafael Zalamena
   409    Mark Stapp
   388    Christian Franke
   387    Daniel Walton
   384    Martin Winter
   365    Donatas Abraitis
```

Now I feel that I can read the code, I can find the root causes faster, I can even review others' code, not that dumb and green as was 2 years ago.

When I started digging into FRRouting (because it was picked by Cumulus Networks) I solved old issues (3-4 years old), there were really lots of them. I managed to solve quite a lot, more to go still, but it's never-ending for a huge and fast-moving project. Of course, I'm not referring [github.com/ansible/ansible](https://github.com/ansible/ansible/), which is I would say brain dead.

What I do now? I participate in the development process more and more. Mostly I contribute to BGP protocol (my favorite one), packaging, and testing environments. Sometimes in other areas (don't have much experience and knowledge yet of other protocols).

Overall my commits during two years (including merge commits of other contributors):
```
% git shortlog --summary --numbered --author='Donatas Abraitis'
   365  Donatas Abraitis
```

Commits owned by me:
```
% git shortlog --summary --numbered --no-merges --author='Donatas Abraitis'
   204  Donatas Abraitis
```

Doing the math one commit every third day. For some people, I know that would be like a full-time job. Comparing with my regular work, two of the most active repositories I contribute are:
```
% git shortlog --summary --numbered --no-merges --since="2 years ago" --author='Donatas Abraitis'
  1148  Donatas Abraitis

% git shortlog --summary --numbered --no-merges --since="2 years ago" --author='Donatas Abraitis'
   355  Donatas Abraitis
```

That sounds like a full-time vs. a part-time job.

I absolutely do not regret that I spent plenty of my free time contributing to FRRouting because that helped me grow as a person and professionally. Also, I started understanding lots of things about how huge open-source projects work, how the deployments run, communication, rules. Third home.

I see that with more time you get more trust from the community, you get valued. At the beginning, I thought I'm very annoying and angry about reviewing others' code, but it seems not, people value that.

We just can't push the bad code, which does not keep requirements, no documentation updates, and testing. That's not a private project which is used by 10 people (what is usually seen in most companies). There is no excuse to push the bad or untested code.

Basically, we are trying to keep requirements like Linux kernel does: checkpatch.pl, clang-formatter, etc.

I remember one day I was tackling a [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) issue at [Hostinger](https://www.hostinger.com/) and wanna check how caching is implemented in dnsmasq because the performance comparing to [tinydns](https://cr.yp.to/djbdns/tinydns.html) is really significant (tinydns 10x faster).

I pulled the source code to dig into the problem I was facing (not the scope of this blog post). Guess what was my first words (direct). "Abandoned university bachelor's code."
Yes, that's the very very notable difference when you look at the code which is community-driven and which is just another one more project.

How could it be better when you commit and even more maintain the world-class project which is used by such big players like Microsoft, Amazon, RedHat, VMWare, Cumulus Networks, etc.? Wait, Cumulus Networks acquired by [NVIDIA](https://blogs.nvidia.com/blog/2020/05/04/nvidia-acquires-cumulus/), that complicates things even more :)

My kids sometimes ask me, why do you work all the time? Well, I answer honestly that I'm not working, I'm learning. It's my hobby, the same as you watching "Nastya", "Roma and Diana", "AcroYoga", etc., riding a bike.

Sometimes doing that at weekends, usually when I have nothing else planned to do. Don't expect you doing that.

I always think about leaving the project, but it's hard, it's like a third family, where you have an amazing community, good practices, super-hero developers, you just learn, you can't leave your home :)

This is not the same as a regular job, you don't get money. But you kinda drive a project which is brilliant. Which is emotionally awesome.

To sum up, being maintainer doesn't mean you have to be pro-level in everything. I formed my opinion that it's enough to be active, responsive, willing to help, learn a lot, dig very deep into the protocols, and of course going wild reading RFCs.
