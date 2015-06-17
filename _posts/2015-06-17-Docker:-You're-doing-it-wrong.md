---
layout: post
title: Docker, You're doing it wrong
categories: [general, docker, ramblings]
tags: [rants, ramblings, docker, things I write on planes]
fullview: true
---
I tend (as many people) to blame a technology for what people actually do with that technology. But think about it, who's to blame for Hiroshima, the A-Bomb? Or the guys from the Manhattan Project?

I know, probably way out of context, and comparing Docker with the A-bomb would definitely get me a bit of heat in certain circles. But here's the thing, this article is not titled "I hate Docker" (must confess the idea crossed my mind), but it's rather "Docker: You're doing it wrong".

Let's begin by saying that containers is not a new idea, they have been around for ages in Solaris, Linux, AIX among others, and soon enough, in Windows!. We actually have to thank Docker for that one.
Docker was definitely innovative about the way they're packaged, deployed, etc. ... It's a great way to run micro services, you'd download this minimal image with nginx, or node, or {insert your language/toolkit/app server here} and run your workload.

Now here's the problem with that, a lot people (ejem... developers) will download an image with a stack ready to go, bundle their script, and tell someone from the ops team "Hey go and deploy this, you don't need to worry about the dependencies". Now a great ops guy, will answer "That's great, but:"


- Where did you get this software from?
- What's inside this magic container?
- Who's going to patch it?
- Who's going to support it?
- What about security?

And so begins a traditional rant, with phrases like "code monkey" and "cable thrower/hardware lover" that could last for hours debating what's the best way to do deployments.

Hey, remember? We're in the DevOps age now, we don't fight over those meaningless things anymore. We do things right, we share tooling and best practices.

So let's be a little more pedagogic about Docker from a System Administrator perspective:


- Over 30% of the images available in the Docker Hub contain vulnerabilities, according to [this report](http://www.infoq.com/news/2015/05/Docker-Image-Vulnerabilities), so where did you get this software from is kind of a big thing.
- Generally, downloading images "just because it's easier" leads to questions like what's that container actually running. Granted, it's you that actually expose the ports open in the container, but what if some weird binary decides to call home?
- If you're about to tell me that your containers are really stateless, and you don't patch and that in CI you create and destroy machines and you don't need to patch, I've two words for you, 'Heartbleed much?'. Try to tell your CIO, or your Tech Manager you don't need to patch.
- Is your image based on an Enterprise distribution? How far has it been tuned? Who do I call when it starts crashing randomly.
- XSA-108 anyone? Remember cloud reboots? Even a mature technology that has been on the market for years is prone to have security issues, so there is no guarantee that something weird running in your container might actually hit a 0-day and start reading memory segments that don't belong to it.

![Vulnerabilities in Docker Images, extracted from http://www.infoq.com/news/2015/05/Docker-Image-Vulnerabilities](/assets/media/docker/dockerhub-official-image001.png "Vulnerabilities in Docker Images, extracted from http://www.infoq.com/news/2015/05/Docker-Image-Vulnerabilities")


Now I don't pretend to discourage you from actually using Docker, again, it's a great tool if you use it wisely. But build your own images!. Understand what you're putting into them. Make it part of your development cycle to actually update them and re-release them. Use it as it was designed, i.e. stateless!.

I know that this might cause flame wars and you are desperately looking for a comments box to express your anger on how I've been trashing Docker. 


- First, I haven't!. Docker is a great tool.
- Second, if you want to correct me on any of the points above, there is enough points to contact me, and I'm happy to rectify my views. 
- Third, feel free to call me a caveman on your own blog, but progress shouldn't compromise security!.
