---
layout: post
title:  Can DevOps Be Outsourced? (And what that means for Rackspace)
date:   2016-08-31
tags: [devops, rackspace]
excerpt: The recent purchase of Rackspace by PE firm Apollo Global Management coincides nicely with the deletion of my company's last Rackspace Cloud server.
redirect_from: /2016/08/31/can-devops-be-outsourced/
---

The recent <a href="http://www.businessinsider.com/rackspace-goes-private-to-focus-on-customer-management-2016-8" target="_blank">purchase of Rackspace</a> by PE firm Apollo Global Management coincides nicely with the deletion of my company's last Rackspace Cloud server. I've long wanted Rackspace to "win" against the likes of Amazon and Microsoft, but their focus on services vs. infrastructure has made good business sense for a while now, so I'm happy they now have more cash to throw at it. Good business sense, however, may not always translate into a working technical solution, as our own experience with Rackspace services suggests.

Back in August of 2014, we had just two developers at GeoStrategies—Matthew, responsible for the frontend, and me, responsible for everything else. We expected to grow quickly, and needed to get our new platform off the ground as soon as possible. We also knew our "snowflake" style of server management would kill any hope of scalability and exacerbate the "it worked in development!" syndrome we were already experiencing. Infrastructure-as-code was the obvious answer.

## In which DevOps turns out to be hard work
At the time, I knew nothing about DevOps and its toolchain. I did know it'd be worth learning, so despite my programming load I put everything on the back burner for a week or two and set out to learn. Some googling led me to rule out Puppet and Chef. They seemed to be losing momentum and I wasn't thrilled to add Ruby to our list of required skills. Ansible and SaltStack were the new hotness, and I like Python, so I decided to give Ansible a shot.

Guess what? Ansible—and all the rest—are tools that help YOU manage infrastructure and configuration. That's right: YOU are still responsible for knowing all about setting up MySQL and MongoDB and Nginx. It's up to YOU to apply patches, lockdown SSH, and keep China on the other side of your firewall. And then YOU still have to figure out how to deploy your app on top of it all. Ansible made me a far more capable sysadmin, but I needed to be a programmer.

## Let's outsource!
Having come to the above realization, I started looking for other options, and stumbled upon Rackspace's "DevOps Automation" service. It sounded _perfect_. Their team would manage our servers, deploy our app, AND build us a Vagrant-based local development environment that exactly mirrored production. It would increase the cost of our project a bit, but I figured having consistently reproduceable deployments would be worth the expense. At any rate, it was a lot less expense than hiring another programmer or a dedicated sysadmin/devops.

Sadly, what started out as a great idea quickly turned into a management nightmare and communication disaster. We still talk about our time with Rackspace—it's the bad experience against which we compare all other bad experiences—but perhaps it wasn't entirely Rackspace's fault. Let me tell you how it started.

## Rackspace, you've got a good sales team

Our experience was at first extremely encouraging. Before we had put down a dime or even discussed pricing, a member of Rackspace's technical sales team flew out to meet us in our own office. His name was Chandra. Chandra and I spent most of the day discussing our requirements in depth, he—Rackspace—bought lunch at the <a href="http://www.dustbowlbrewing.com/" target="_blank">Dust Bowl</a> for several of us, and I said goodbye to him highly confident that Rackspace understood every detail of our DevOps needs and would bend over backwards to take care of us.

The next day Chandra emailed the minutes of our meeting to us and to our account manager at Rackspace. They were detailed, accurate, and reflected a level of careful attention that bode well for our ongoing relationship. That was the last we heard of Chandra, and our last wholly positive experience with Rackspace.

We spent the next couple weeks bouncing from conference call to conference call reminding people of Chandra's notes and practically begging for an estimate. We finally learned that an initial buildout through their DevOps Advisory service would cost the same as several months of full DevOps Automation, so we chose the latter. They would use Chef to manage our servers and deploy our app, and we could get back to writing code. We had outsourced our DevOps.

The hiccups started coming right away. The DevOps team sent us a questionnaire covering old ground in much less detail—they apparently didn't know about Chandra. They then took a few weeks to setup a basic stack of Nginx+PHP+MySQL. ("Why?", I wondered. "That's what it would have taken me in Ansible.") The stack they set up wasn't even close to meeting our needs, so we spent some more time reiterating our requirements over email, on the phone, and in their ticketing system. The saga had begun.

## I do not think it means what you think it means
Our primary difficulty was with communication, both from our team to Rackspace's, and within Rackspace's team itself. The Lost Notes of Chandra became emblematic of our entire experience. I'd talk to someone on the phone, they'd begin making a change, not quite finish it, and the next shift would have no idea what had happened. Or I'd open a detailed bug report on their ticketing system, see someone understand it and begin to make progress, and then have someone else comment that it was fixed, even though it clearly remained broken—he obviously hadn't read most of the ticket. One step in particular—simply exposing environment variables to our app—took more than a month to implement. I'm still not sure how that's possible. Someway, somehow, they just weren't talking to each other.

A second difficulty was perhaps more fundamental: loss of agility. More fundamental than communication, you ask? Yes, I believe so. Our communication problems, bad as they were, could have been avoided, and indeed I'm quite sure that Rackspace has improved on that point substantially. The loss of agility, however, was the natural and unavoidable consequence of ceding all powers of configuration to a team outside our office.

I was given a command to run for deploying our app, and that was about it. Need to point to a different Git tag? Talk to Rackspace. Need to add/update an environment variable? Talk to Rackspace. Need to setup another microservice or Cron job? Talk to Rackspace. The overhead of passing every menial task to another team—and having to explain them all!—effectively prevented a wide range of tinkering on our app. We deployed code generally once a week, and about once a week I wished we had just a little more control over the process.

Do you see what had happened there? We bought "DevOps", and got exactly what DevOps was meant to replace.

## Can It Be Done?
DevOps started as a reaction against the ancient tension between developer and sysadmin. With different responsibilities, each team naturally has its own priorities, practices, and viewpoints. If we can more closely marry the two by improving communication and sharing responsibilities, we'll substantially minimize a whole category of risk in software development. And yes, that's the opposite of outsourcing.

When you hire an outside team, you may buy all the skills your programmers lack, and you may get infrastructure-as-code at an unbeatable price, but you may also find it's not DevOps. Deployments may be even harder than before, and change may move at an even more glacial pace, and if that happens you need to go back the way you came. I'm not saying it absolutely can't be done, but I am saying it might not be worth trying.

So where does this leave Rackspace (and Apollo Management)? I really don't know. I suppose there will always be plenty of cloud-related services to offer, but I can't help thinking there's a pretty hard ceiling they're going to hit as developers become more versed in DevOps and as more and more automation is built every year to take the pain out of it. Regardless, I wish them luck as they seek to expand in that sector. But just as much, I wish you luck as you tackle DevOps on your own.
