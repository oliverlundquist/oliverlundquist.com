---
layout:           post
title:            "How to run a local offline development Pusher server."
date:             2018-05-16 14:25:10 +0900
last_modified_at: 2018-05-16 15:37:20 +0900
tags:             [pusher, websocket, websockets, local, dev, development, server, offline]
introduction:     "In this article, I will go through how I setup a local Pusher websocket server on my local machine, this is something that's useful for getting offline support or save Pusher requests."
---

So I had 30 minutes before I had to board my 11 hours flight from Tokyo to Copenhagen. I had to work on a new websocket based functionality that I was building at the moment that was using Pusher. Not being able to access the internet and Pusher's servers would make it difficult to work on this new feature. So I started to look into if Pusher had a local development server available so that I could use it offline.

After a couple of quick google searches, I found that there is a project called [pusher-fake](https://github.com/tristandunn/pusher-fake){:target="_blank"} and that someone built a [Dockerfile](https://github.com/gregeinfrank/docker-pusher-fake){:target="_blank"} for this project. It is not released or maintained by Pusher themselves but it seems to have all the functionality which is just what I was looking for.

I needed a couple of customizations so I quickly built my own version of the image and created a basic proof of concept to see if I could get this working. After creating the docker image I created a quick PHP script and a simple JavaScript frontend client that is listening for messages from the PHP script. The PHP script is (located [here](https://github.com/oliverlundquist/pusher-local-dev-server/blob/6a50f4b315e22f3237ba60a918c87f97a7a5d68c/pusher.php){:target="_blank"}) and the frontend client is (located [here](https://github.com/oliverlundquist/pusher-local-dev-server/blob/6a50f4b315e22f3237ba60a918c87f97a7a5d68c/index.html){:target="_blank"}) if you would like to check it out.

The backend PHP script sends messages to the local Pusher server which then broadcasts it to all the frontend clients that are connected to the Pusher local dev server. Every time the frontend client receives a message it changes the background color and should look something like [this](/assets/pusher-local-dev-server-demo.gif){:target="_blank"}.

To run the demo application yourself simply follow these 4 steps:

{% highlight plaintext %}
git clone git@github.com:oliverlundquist/pusher-local-dev-server.git
cd pusher-local-dev-server
docker-compose up
open index.html (or double-click the index.html file to open it in your browser)
{% endhighlight %}

That’s it for this time, setting up a local dev Pusher server is great if you are on a limited plan and do not want to waste requests on your Pusher account while developing applications on your local machine. It’s also useful if you like me need offline support for your application that is using Pusher.

Until next time, have a good one!

Don’t forget to check out the full repository here:
- [https://github.com/oliverlundquist/pusher-local-dev-server](https://github.com/oliverlundquist/pusher-local-dev-server).
