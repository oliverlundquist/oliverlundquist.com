---
layout:           post
title:            "Chrome doesn't resolve localhost TLDs to my local IP address."
date:             2018-02-26 19:06:00 +0900
last_modified_at: 2018-02-26 19:54:00 +0900
tags:             [google, chrome, localhost, domain, tld, resolve, ip, local, development, environment, virtualbox, vm]
introduction:     "Since Chrome 63 I switched an old project to the .localhost top-level domain and noticed that it stopped resolving my IP to my VirtualBox VM. After trying a bunch of different things I finally found out why it wasn't working."
---

Today I picked up an older project that I haven't been working on for a while. It is more or less built with the presumption that the app is running on the service.dev domain. However, since Chrome 63 was released .dev top-level domains are now forced to and automatically redirected to the HTTPS protocol. If you have your SSL certificate setup and in place on your dev server this will typically not make any difference but if you like me have been running your app in your development environment mainly on HTTP you will have to make some adjustments.

Either you add an SSL certificate to your development web server and setup HTTPS or you switch TLD to something that is actually targeted for local development, like the .localhost top-level domain. Having limited time I thought that I would be clever and went ahead simply switching the app to run on the service.localhost domain instead.

I had an entry in `/etc/hosts` that pointed service.dev to the custom IP of my VirtualBox VM which has been functioning great all this time. I quickly switched this to service.localhost and tried to access my app. To my surprise I got a Connection Refused error in Google Chrome.

Now, some of you might have figured this one out already, but I started to desperately looking for ways to clear my DNS cache in Google Chrome since I had confirmed that a DNS lookup and a PING request was working just fine and resolved my local IP to my VirtualBox VM just fine.

I tried the following things:
- Clearing DNS host cache in `chrome://net-internals/#dns`
- Flushing socket pools in `chrome://net-internals/#sockets`
- Clearing the MacOS DNS cache with `sudo killall -HUP mDNSResponder`
- Turning off the "Use a prediction service to load pages more quickly" setting in `chrome://settings`
- Installing Google Chrome Canary
- Rebooting my computer countless of times

But none of these worked. Now I started to get really desperate and started tweaking the config of my dnsmasq setup in my local environment to try to force Google Chrome to pick up the new DNS settings on my machine. I was convinced that old DNS cache in Google Chrome was the issue since I got the expected results from DNS lookups and PINGs that I was executing from my command line.

After taking a break, doing some exercise and having some dinner I suddenly remembered that Google Chrome automatically resolves all .localhost TLDs to the loopback interface or `127.0.0.1`. I must say that I felt rather embarrassed when this finally occurred to me after a rather long trial and error session, especially since I told a colleague the same thing just a couple of weeks ago when he was in a similar situation.

Nevertheless, the feeling of figuring it out after being stuck for half a day is really refreshing and is what keeps me going as a web developer. This time I felt so relieved that I had to channel all this positive energy into a blog article.

The best solution in the end for me and this project was to keep the .dev TLD and instead create a self-signed certificate and install that in my local development web server to enable HTTPS support instead of switching to using the .localhost top-level domain. However, you could also switch to another TLD than .dev if you would like to keep a similar setup without HTTPS like before.

I wrote about how to create a wildcard SSL certificate for your local development environment in this [blog](/2018/02/26/setup-ssl-and-https-in-your-local-environment.html)

If you want to read more about reserved top-level DNS names check out this RFC: [https://tools.ietf.org/html/rfc2606](https://tools.ietf.org/html/rfc2606)
