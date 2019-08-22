---
layout: post
title: Became maintainer of FRRouting
categories:
- blog
---

This is the main reason I stopped (or less) blogging because I’m more interested in how the internet works and give my full fingers improving the internet overall.

At first, it sounds amazing on the title. Yes, that's really cool and motivating. Secondly, you have to pay more attention to what you do as a developer plus review others code where you have some degree to do it, continue reading the source code, help the community to adopt the product for their use.

I started contributing to FRRouting open-source project around one and a half year ago. I just went over existing issues and tried reducing them (solving). In the beginning, I felt like I got a new job. Setup development environment, new tools, formaters, linters, strict requirements, similar to the Linux kernel's workflow, etc. It was fun and I learned lots of really new stuff, met new especially very very smart people.

After a year I was promoted being the maintainer of FRRouting project.

![](/images/frr-maintainer.png)

What does it mean?

Maintaining a huge world-class project is a huge responsibility because you can quickly lose your trust. We have weekly virtual meetings to discuss the current issues, pull requests and then decide who takes which part or who will review something existing. We have Slack channels where discuss some only related things. No rush. It’s always better to have a slow loop rather than going through the fast loop and fixing things again and again.

The lifespan the PR takes to be reviewed sometimes takes even months. Code reviewers have to actually look at the code, they can't just hope the community will somehow perform a proper review. A project also cannot rely just on tools, they will never do the full job. Proper code review takes a lot of time, and it needs to be done by experienced developers.

Responsibility comes when you have to review or fix parts you already changed/improved. I always had a fear of commenting, merging and discussing the stuff you are not the best one comparing with others. With time I became more comfortable being part of the project.

Noticed an interesting issue and wanted trying it as a challenge? Let’s assign self and start the journey. Sometimes I have no clue if I can or can’t implement assigned issue at all, but it's even more challenging than a regular job because you must write good code, you must keep styling, testing and other requirements in place. In regular jobs, you mostly can ignore that while here no. Comparing to a permanent job, code review could be more strict in open source projects because you have to keep all the things unified.

Companies require to push features or bugfixes as fast as possible even without carrying about the performance, quality and other factors. Just push forward. In open-source, things differ. Your commits could stay even months not merged. And that's normal. Why? Because of both: the quality and performance.

My one of the favorite phrase Linus said:
>The New Linus: "I really like you as a person and all. But that code will only get committed over my dead body.

Again, maintaining a global project doesn’t mean you have full control and do what you want. For instance, implement BGP version capability, but it’s absolutely not worth doing this without published draft/RFC.

So I created a [draft](https://www.ietf.org/id/draft-abraitis-bgp-version-capability-01.txt), then we discussed in [IDR mailing](https://mailarchive.ietf.org/arch/browse/idr) list about all the options and culprits. Some were agreeing with me, some not, but it's absolutely fine to be rejected eventually. However, writing draft/RFC is really not the funniest thing to do, because at first it must be written in XML, then you must pass the validations (syntax, grammar, paragraphs, references, etc.), discuss with the community if it’s worth, code a working example/prototype as a reference.

### Fun facts

* Google (rumors that Facebook as well) uses FRRouting internally;
* I was invited to a [job interview](https://teltonika.lt), then I asked them why did you invite me? "We saw you are a maintainer of a project we use in our devices for routing";
* The best thing that happened apart from the birth of a child - huge motivation to move forward.
