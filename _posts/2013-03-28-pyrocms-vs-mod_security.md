---
layout: post
title:  PyroCMS vs. mod_security
date:   2013-03-28 17:32:12
tags: [apache,pyrocms,mod_security,sql injection]
excerpt: Upon clicking the last button on the last step of the installation, the whole thing fell apart.
---
<section class="callout secondary">
<h3>Update 11 April 2013</h3>
<p>I just ran into this problem again. Yesterday, InMotion Hosting automatically overwrote my edited mod_security rules. I chatted with Dustin P at IMH who informed me that, apart from completely disabling mod_security, there is no way to protect my edited rules from being overwritten whenever the IMH team applies updates. I also realized that these rules (obviously) apply to their shared hosting too. Since unlike mod_sec <a href="http://forums.cpanel.net/f5/mod_security2-how-disable-via-htaccess-72207.html">mod_sec2 cannot be disabled in an .htaccess file</a>, on shared hosting with IMH you must either find a new host or modify PyroCMS.</p>

<p>Conclusion: even with a VPS, InMotion Hosting may not be the best place to host PyroCMS.</p>
</section>

I've recently been delving into <a title="PyroCMS" href="https://www.pyrocms.com/" target="_blank">PyroCMS</a>. It's lightweight, easy to use, and 100% customizable. It's built on a framework I like (<a href="http://ellislab.com/codeigniter" target="_blank">CodeIgniter</a>) and transitioning to a framework I'm learning to love (<a href="http://laravel.com/" target="_blank">Laravel</a>). If you haven't used it yet, I highly recommend giving it a shot. That said, my first installation attempt almost convinced me to look elsewhere for a more sturdy CMS.

Upon clicking the last button on the last step of the installation, the whole thing fell apart. I went to the front page to see if anything was there, and all I saw was completely unstyled HTML. Assuming the template hadn't installed properly, I open my developer tools to look for 404s. Obviously the CSS wasn't loading, but then neither was anything else, and these weren't 404s. Every request to the server was returning "406 Not Acceptable." Ok, so somehow the files are in the wrong encoding, I think. Nope, they're UTF-8. Next I went to check the .htaccess file for anything suspicious. But there was no .htaccess file. As far as I could tell, PyroCMS was not setting any server directives at all, so it couldn't be what was causing the 406.

At this point, I couldn't tell what was wrong, so I decided to delete PyroCMS and start over, paying much closer attention next time around. To my horror, deleting PyroCMS <em>did nothing</em>. I deleted the entire directory, created a new directory, and put an empty index.html in it, and I <em>still</em> got the 406. Opening a new browser, I quickly realized there had to be a bad cookie or something set by the installation because my other browser received a friendly 200 OK. I cleared my cookies, reinstalled Pyro, and again got the 406.

After much experimentation, I found the root of the problem. The PyroCMS installer sets a cookie containing various data including the admin user's information such as email and password. As long as the field "user_password" was set, my server returned an error. If I removed that field from the cookie, everything was fine.

Turns out this is an uncommon but known difficulty in the PyroCMS installation. (See <a href="https://github.com/pyrocms/pyrocms/issues/630" target="_blank">#630</a> and <a href="https://github.com/pyrocms/pyrocms/issues/1927" target="_blank">#1927</a>.) I say "difficulty" because it's not really a bug. Apache's mod\_security is often configured to deny requests (read: reply with 406) which contain anything resembling an SQL injection, and "user\_password" can be one such string. For every request made after the last installation step, Apache saw "user_password" in the incoming cookie, decided there was an SQL injection in progress, and returned a 406.

How do you fix this? In my case it was easy since I was on a VPS from InMotion Hosting. I ran `grep "user_password" *`  in /usr/local/apache/conf, edited the offending lines, and restarted Apache. If you're on shared hosting and run into this problem, you unfortunately have only two options: either modify PyroCMS to use something like "user_pass" during installation, or find a new host.

While installation errors tend to leave a bad taste in my mouth, this one was mostly the fault of overzealous sysadmins. With my installation over and PyroCMS exonerated, I'm finding it quite an excellent addition to my toolbox.