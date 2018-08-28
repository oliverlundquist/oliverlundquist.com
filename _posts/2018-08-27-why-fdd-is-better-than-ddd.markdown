---
layout:           post
title:            "Why FDD is better than DDD in Laravel applications."
date:             2018-08-27 14:25:10 +0900
last_modified_at: 2018-08-27 15:37:20 +0900
tags:             [fdd, ddd, cdd, laravel, architecture, agile, productivity, motivation]
introduction:     "Feature Driven Design (FDD) can drastically improve your development flow by creating shorter iterations and quicker feedback loops which results in better software and a happier more motivated team."
---

This time around, I've been working on a Laravel project adopting a lot of the best practices from Eric Evans' Domain Driven Design (DDD). After 6 months in, I started wondering if this really is a good approach for Laravel and other modern applications. I got this feeling because most of the code I was writing was to either abstract or work around the core parts of the framework instead of utilizing its power in a more direct manner.

Many times I would create factories and translation logic translating attributes from domain objects to Eloquent models, which could be persisted to the data store and vice versa for when getting data from the database. I wrote QueueService interfaces that were implemented in the infrastructure layer with only one line of code calling the Laravel dispatch() function. The IoC bindings, the interfaces and factories I wrote were simply to abstract the functionality of the framework, to not use it directly since that would violate the DDD domain principle.

Creating interfaces to abstract logic is considered good practice and few would argue against that they make it easier to change the implementation behind the interface, it's easier to code against an interface. However, what's important to not forget is to think about how often you're actually going to switch the data store engine or how often you're going to change the framework you used to build your application. Most of the time I'd say that the data store implementation and framework never change during the lifetime of an application, even if it would, you would probably need to rewrite a big chunk of the application anyway.

I'm a big fan of YAGNI (You Ain't Gonna Need It) and I'm a firm believer of not creating stuff that you do not need. Therefore, I didn't find it useful creating this overhead in my Laravel project just because it would theoretically make it easier to one day switch my MySQL database to something else, which would probably never happen.

Looking around, I found the methodology Feature Driven Design (FDD) which I think solves a lot of these issues. Unfortunately, it hasn't got as much attention as DDD, but I think it is a better fit for modern applications using modern frameworks. In this open-source software (OSS) era that we are living in right now, where you can find any type of functionality or library as a repository on GitHub, we write less and less boilerplate code. Uploading a file to AWS S3 or storing an object in a database is more or less just one line of code.

I believe this is really an awesome era to develop software because applications can be built so much faster and with much less effort. That makes it much easier to focus on your application and what you want it to be able to achieve instead of thinking about how to connect to the database or how to route the incoming traffic to a controller or view to get some output to the screen.

So, I think it makes sense to embrace these powerful tools and not worry if we are violating a design pattern or if we're going to change the database in 5 years from now. As long as the application is coupled with a good amount of automated tests, it doesn't matter how many interfaces we put in front of our Eloquent models or how small we make the surface of the Laravel Query Builder, if the application works, it works.

I've already covered some parts in the introduction but here are my top 4 points why I think FDD could be a better choice for applications built on modern frameworks with built-in ORMs, queues and other infrastructure logic.

### Top 4 reasons why I believe FDD is superior to DDD in modern applications.

**The focus is on what the application should be able to do.**

Designing domains in DDD are difficult, designing and drawing boundaries of models, thinking about things like if you should make a model an aggregated root or not, can cost a lot of initial cost and overhead which makes the project slower to get going.

Instead, focusing on what the application should be able to do and encapsulating that logic into feature objects will make it easier and quicker for you to start once you have all the requirements of the features that the application will need.

**Faster iterations and shorter feedback loops.**

Have you ever started on a new project really excited, just to see the motivation drain from you once the project hits a certain size? In application development, we need short development iterations and quick feedback loops to keep the direction of the project spot on and the motivation of the team soaring. Remember, you're not building software for yourself.

Therefore, focusing on the features as features instead of working on keeping the domain as plain as possible and working around your framework and OSS toolset to follow the DDD architecture often create overhead and slows down the development velocity.

**It scales in larger teams and is also good for asynchronous development**

FDD focuses on the features and not the domain of an application and therefore scales very well among larger teams. Features are encapsulated in feature objects which can be worked on individually. There is no need to wait for some other developer to finish the design and coding of their domain models for you to proceed with your work. Because features can proceed individually and aren't dependent on each other it's also a great methodology to use in teams spread out across multiple time zones.

**Revise and add new features without being a domain expert**

If you're new to a certain domain or if you're a new developer in a team, it's often hard to revise or add new code to an application if you're not used to the domain. Much like CDD (Component Driven Development), in FDD, logic is separated into encapsulated feature objects. This makes it easy to find the logic you're looking for and it's also easy to add a new feature since it's not dependent on existing logic. This encapsulation also makes it easier to try new concepts and infrastructure on a few features.

### How to start developing with FDD?

Start by making a list of the features needed for your application. Once that's done, just create one class per feature and dump them into a folder wherever you like. The point is to keep the focus on the features themselves and not where you put your files.

Let's say you're building a t-shirt app with a t-shirt list page and a t-shirt details page which you're redirected to after clicking on one of the t-shirts. Firstly, we'll need to create a GetTShirts and a GetTShirtDetails class. We probably also want to be able to filter t-shirts on the list page, so we'll need a GetTShirtsByFilter class.

Thanks to modern frameworks with built-in ORMs and to OSS SDKs much of the operations we need like storing/retrieving images from AWS S3, fetching objects from a data store or sending a mail to a customer is just one line of code away. This will make your feature objects slim and there won't be much duplicated code since you're just making a call to the framework ORM or third-party SDK.

However, once the project gets larger and if you notice that you for example always make the same currency conversion inside many of your feature objects, it makes sense to break out that logic to a service. But don't waste time worrying about duplicated code if it's not there, moving code out to services will come naturally when the time for it has come.

### How to structure code?

In FDD, the central part of the design are the features, not the domain. I believe it's good to keep a flexible folder structure and not spend too much time thinking about where to put your files. That said, without showing an example of how things could be structured it could be hard to wrap your head around it, so here we go:

{% highlight plaintext %}
ExcellentFabric // company namespace, could contain many apps
├── TShirtStore // app namespace
    ├── Features
    │   └── TShirts
    │       ├── GetTShirtDetails.php
    │       ├── GetTShirts.php
    │       └── GetTShirtsByFilters.php
    ├── Infrastructure
    │   ├── Eloquent
    │   │   └── TShirtsEloquent.php
    │   └── S3
    │       └── TShirtImagesDAO.php
    ├── Services
    │   └── Currency
    │       └── CurrencyConverter.php
    ├── Tests
    │   ├── Features
    │   │   └── TShirts
    │   │       └── GetTShirtByIdTest.php
    │   └── Unit
    │       └── TShirts
    │           └── Services
    │               └── Currency
    │                   └── CurrencyConverterTest.php
    └── UI
        ├── Api
        │   └── Controllers
        │       └── TShirts
        │           └── GetTShirtsController.php
        ├── DTOs
        │   └── ProductIndexViewFiltersDTO.php
        ├── Middleware
        │   └── TokenVerifier.php
        └── Web
            ├── Controllers
            │   └── TShirts
            │       ├── TShirtDetailsController.php
            │       └── TShirtIndexController.php
            └── Views
                └── TShirts
                    ├── TShirtDetailsView.blade.php
                    └── TShirtIndexView.blade.php
{% endhighlight %}

I hope that this was an interesting read and that it gave you the inspiration to give FDD a try. I would love your input and thoughts on this type of architecture, pros and cons, so please do post a comment down below.

Until next time, have a good one!
