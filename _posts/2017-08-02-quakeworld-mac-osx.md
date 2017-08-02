---
layout: post
title: Running QuakeWorld under OSX
categories:
- blog
---

Already 15 years passed when I played [QuakeWorld](https://en.wikipedia.org/wiki/QuakeWorld) and I feel too much nostalgic remembering those days. QuakeWorld (aka. Quake1) was released in 1996. It celebrates 21 this year and I tried to go back to 2002-2008 (it was the time I played QW quite professional). I must say, that well-known players such like Milton, Locktar are still on the track. 

I always dreamed about attending to [QHLAN](http://qhlan.org/) or [QuakeCon](http://www.quakecon.org) but never did it because of bad time (school, studies). Right now I have such an opportunity, thus I'm planning to attend QHLAN and/or QuakeCon next year! Not to win(?) something but to meet QuakeWorld legends.

I remember my hardware setup when the movement was really smooth. Those days most important components were mouse and monitor. I used [Logitech MX300](https://images10.newegg.com/NeweggImage/ProductImage/26-104-127-03.jpg), [Logitech MX518](https://images-na.ssl-images-amazon.com/images/I/91YmmYMH40L._SL1500_.jpg), [Razer Boomslang](https://www.techspot.com/images/products/mice/org/3_452303283_o.jpg), [Razer Lachesis](https://d3gqasl9vmjfd8.cloudfront.net/83d7f1ef-1902-4ed1-b0f6-f83cebd39e6c.jpg) mouses, 100Hz CRT monitor (which means 100fps), gaming mouse pads such as [Razer Mantis](https://76.my/Malaysia/razer-mantis-control-precision-mousing-surface-ckmultimedia-1605-19-ckmultimedia@4.jpg), [Razer Pro](https://shop.cyes.nl/images/product_images/popup_images/razer-destructor-wit-mousepad-0-281.jpg), thus I do not accept to be worse than in 2002. But it is. I will tell you the story.

Ok, so today we count 2017 and I have what I have. MacbookPro 15" Retina (60Hz). Boys are growing up, hence toys are getting more expensive. I installed [ezQuake](https://ezquake.github.io/) client for OSX, copied back my old great stuff from `id1` and `qw` directories and let's frag. By the way, you can find my configuration (updated) file [here](http://donatas.net/games/ton.cfg).

First moments were more than terrible, the movement looks like moving with the vibrator inside, the frame rate is awful. The feeling is like playing with PentiumII MX200 (my first x86 computer) and without OpenGL.

The first aid is `cl_maxfps`, but it actually wasn't.

With macOS Siera predecessors (Yosemite, Maverick) it was possible to turn off vertical synchronization for GPU. With Siera release, it's not possible. Fortunately, ezQuake has `vid_vsync` switch to on/off this option in the client, not in the driver. It's always better to define this limit to value based on the refresh rate of your monitor (60, 100, 125, 144, ...). Do not set `cl_maxfps` to 0, you will why later in this post.

Because I play with MacbookPro, I have 60fps, thus I set `cl_maxfps` to 60 and I have a lag between frames about `3-10ms` with `vid_vsync 0`. I turned this on because by default it's off. It's a little bit better, because of eliminated v-sync lag which dropped to `0ms`. You can verify the lag by typing `show vidlag` in the console.

![vid_vsync_0](/images/vid_vsync_0.png)
![vid_vsync_1](/images/vid_vsync_1.png)

I read ezQuake documentation and found some really interesting configuration `cl_vsync_lag_fix` which should eliminate the lag. It dropped to `1-2ms` with v-sync turned off, but way too far to be `0`.

Well, I set `cl_maxfps 60`, `vid_vsync 1`. I have no lag, but still not smooth, let's dig more into this stuff. And here `cl_physfps` option comes into play. This is usually set to 77fps on almost all servers. My `cl_maxfps` is 60 (which is a hard cap), the server is running with 77. Fortunately, it's allowed to override this value independently by setting `cl_physfps` to 60.

This is the source code snippet which controls `cl_physfps`:
```
static double MinPhysFrameTime (void)
{
        float fpscap = (cl.maxfps ? cl.maxfps : 72.0);
        float physfps = ((cl.spectator && !cls.demoplayback) ? cl_physfps_spectator.value : cl_physfps.value);

        if (cls.demoplayback)
                return 0;

        if (physfps)
                fpscap = min(fpscap, physfps);

        fpscap = max(fpscap, 10);

        return 1 / fpscap;
}
```

According to this snippet, minimum value should be elected like `min(cl_maxfps, cl_physfps)`, but in the real world in doesn't work. Why? It looks like this works only if `cl_independentPhysics` is turned on:

```
void CL_Frame (double time)
{
        ...
        if (cl_independentPhysics.value != 0)
        {
                double minphysframetime = MinPhysFrameTime();
        ...
```

I set `cl_independentPhysics` to 1, capped `cl_physfps` and `cl_maxfps` to 60 and it's smooth!

#### Conclusion

Quake1 is not dead, it will never be changed with something new! There were few attempts: Quake2, Quake3, Quake4, QuakeChampion. I played Quake1, Quake2, Quake3 and I can confirm that with every new Quake version the gameplay is getting boring and boring. I don't know how it's going with QuakeChampion, but I'm pretty sure it's not comparable with Quake1 anymore! The good news with QuakeChampion is that it inherited [bunnyjumps](http://quake.wikia.com/wiki/Bunny_Hopping) from Quake1.

Happy fragging!

