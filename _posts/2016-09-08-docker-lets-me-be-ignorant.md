---
layout: post
title:  Docker Lets Me Be Ignorant
date:   2016-09-08
tags: [devops, docker, microservices]
excerpt: As a full-time full-stack developer, I have enough on my plate. I'm sure you're in the same boat.
---

As a full-time full-stack developer in a tiny team, I have enough on my plate. I most definitely do not want to add "resident ops guy and Cloud Engineer" to my list of titles. Though I love learning, careful ignorance of extraneous details is a necessary skill if I am to remain effective in my primary role. I [wrote previously]({{ site.baseurl }}/2016/can-devops-be-outsourced/) how this reasoning lead me down the mistaken path of outsourcing our DevOps.

What I didn't tell you was that, however bad our communication problems were with Rackspace, our real problem was with Chef and its traditional style of DevOps. Had we used Docker from the beginning, many of the other difficulties we experienced simply wouldn't have happened.

If you're trying to decide between Docker and more a traditional DevOps tool like Puppet/Chef/Ansible/SaltStack, this post is for you.

## Separation of Concerns – Or Not
We're all familiar with the closely related concepts referred to as "Separation of Concerns", "Single Responsibility Principle", and "Principle of Least Knowledge". When you follow these principles, you get two main advantages:

1. Your system is more robust because changes or hiccups in one module aren't likely to ripple over into another.
2. Your progress is faster and code better because experts need work only in their area of expertise. Delineated responsibilities in code translates naturally into the real world.

These are exactly what we _didn't_ have with Chef.

The Rackspace DevOps team managed for us a single codebase of Chef cookbooks, recipes, etc. This codebase had two distinct responsibilities: to configure our servers and network, and to deploy our app. Except, they weren't really distinct.

Setting up the firewall was part of the same process as installing MySQL, which of course required doing service discovery, which then needed to send the resulting environment variables to our app, which naturally meant configuring PHP-FPM, installing various PHP extensions, and pulling down our code. Rather than distinct responsibilities, we had a continuum–a long gradient where each step was closely related to the next. Where their servers ended and our app began became a pretty fuzzy line.

The result? Fragile code and slow development. (By "code" I mean the Chef recipes, etc.) I'd ask for a change in one area and inevitably something unrelated would break. But the greater difficulty lay in our dependence on Rackspace to do _everything_.

## If we could just do it ourselves...
Since everything outside our app was equally part of a monolithic Chef process, Rackspace and Chef had to know everything. The experts on servers also had to be experts on our app. And that is why Chef couldn't work for us.

The fundamental problem with traditional, Chef-style DevOps is the inability to isolate the application from the environment in which it runs, and the consequent inability to allocate experts to their areas of expertise.

I can't tell you how many times I wished I could edit our PHP configuration or add an extension without having to know anything else. If you asked the Rackspace DevOps team, I'm sure they'd agree.

Let's say you're an expert house-builder but have only a passing knowledge of roads and sewers. What if I tell you your house can't be built unless you first build the rest of the neighborhood? This was the analogy that crystalized in my mind as we neared the end of our time with Chef and Rackspace. If only there were a way I could build perfect houses ahead of time without knowing the neighborhood, and then let someone else build the roads and connect the plumbing....

## Did you say, "build once, deploy anywhere"?
That's exactly what Docker is for: to encapsulate your code in an opaque box. It's like a class or a module in OOP. I have 100% control over my app and its immediate dependencies, and everything else needs to know only the interface I expose. Just as importantly, I don't need to know all about the environment in which it's deployed. Instead of all the back and forth and code fragility, here's how it could've gone down at Rackspace:

> **Josh:** Hey guys, I've got two containers: one that needs to accept public traffic on port 80, and the other that listens on 9000 only within our private network. Can you deploy those for me?  
> **Rackspace:** Sure thing!  
> **Josh:** Oh, and the first one needs access to the second. Currently it expects an environment variable SERVICE_TWO_IP.  
> **Rackspace:** Yeah, no problem. We'll just pass that variable to the first container when it starts.

Obviously we have more than two containers, but that doesn't even matter. Whoever's running them--whether there are 2 or 2000–just needs to setup the network correctly and run service discovery. Those are two surprisingly complex subjects, but now I can let someone else do it without giving up a single iota of control over my app's dependencies.

## So much more than a fad
Whatever you may think of microservices in general and Docker in particular, splitting up responsibilities and knowledge should be one of your highest goals. The truth is that breaking your app into little pieces does increase overall complexity, and Docker itself can be a real bear at times, but the value you gain by distinguishing your app from its environment will make up for it a hundred times over.

If you want to be a high-performing non-sysadmin, don't try to be a sysadmin. Put your app in a container and let someone else run it. Allow yourself to be ignorant of what you don't need to know.
