---
layout:           post
title:            "How to setup Domain Driven Design (DDD) in a Laravel app."
date:             2018-03-20 09:25:10 +0900
last_modified_at: 2018-03-20 09:37:20 +0900
tags:             [laravel, ddd, design, structure, layers, ui, application, domain, infrastructure]
introduction:     "There are many examples of how to use DDD with Laravel on the internet but I believe that this is the cleanest and effective way to structure your DDD code when using Laravel."
---

There has been a lot of discussion about DDD or Domain Driven Design and how we can implement it in our codebase at work. I had some conversations with a colleague that is well read on the DDD topic and after making some tweaks together I integrated that structure into a Laravel app which I will introduce in this article.

DDD is a way of designing your code structure by thinking of your domain or business logic as the center part of your application. From that center part, you branch out your codebase into layers kind of like an onion structure.

The most common layers in DDD are UI, Application, Domain, and Infrastructure.
There are some basic rules to what each of these layers should contain and recommendations on how the logic should flow between these layers. However, these rules do not and cannot be applied in all situations which makes simple tasks as where to put files a bit tricky in DDD.

As a developer, you have to understand the domain and its models to be able to structure the code well. Therefore the communication between developers and other non-developers and managers within an organization has to be crystal clear.

In the DDD world of things, this is called the "Ubiquitous Language" and aims to make the domain or business logic as clear as possible and separate from other more techy parts of your codebase so that it is easier to reason about.

It is actually recommended that the domain only contain plain objects and primitives of the programming language that you are using and not logic or code from a framework or other dependencies.

Before showing the DDD folder structure of the Laravel app, I will make a quick rundown of the four layers in DDD. Imagine that the data flows from top to bottom through these layers in your application.

#### The UI Layer

This is where you put all your entrances and doors into your application.
These entrances could be command-line commands, API endpoints or web interfaces that return HTML data.

#### The Application Layer

In this layer, you tie together all the data flowing in from your user interfaces (CLI, API, Web) with your domain layer.
This layer should be kept as thin as possible and should not contain any real logic.

The application layer is really important because it acts like a cushion between the UI and the Domain layer so that the Domain is not dependent directly on one user interface which makes it easier to add new user interfaces as your application gets bigger.

#### The Domain Layer

The domain layer is where we put all our business logic, this is what you are really building. Think of this as what other non-developers in your organization care about. They do now really care how you run a queue job or what type of data store you get your data from, what they care about is that the application follows the rules and regulations set up in the organization's business logic.

#### The Infrastructure Layer

The last and deepest layer in the architecture is the infrastructure layer. This layer connects to data stores and external services and is all the logic that is related to how you gather data for your domain.

The domain layer should be totally ignorant when it comes to how the infrastructure layer gathers all the necessary data, this also makes it easier for the application to switch data store and external services as you go.

Now you should have a better understanding of each of these layers, let me finish off this article by posting the folder structure that I think is optimal if you would like to structure your Laravel app in a DDD manner.

{% highlight plaintext %}
.
├── Application
│   ├── Providers
│   │   ├── JanitorServiceProvider.php
│   │   └── LaneServiceProvider.php
│   └── Service
│       ├── Janitor
│       │   ├── Cleaning
│       │   │   ├── WipeCafeteriaCoffeeTableRequest.php
│       │   │   └── WipeCafeteriaCoffeeTableService.php
│       │   └── Maintenance
│       │       ├── NextPinsetterMaintenanceDateService.php
│       │       ├── RequestPinsetterMaintenanceRequest.php
│       │       └── RequestPinsetterMaintenanceService.php
│       └── Player
│           ├── ThrowBowlingBallRequest.php
│           └── ThrowBowlingBallService.php
├── Domain
│   ├── Exception
│   │   ├── JanitorOffHoursException.php
│   │   └── MaintenanceAlreadyOnItsWayException.php
│   ├── Model
│   │   ├── Janitor
│   │   │   └── JanitorAggregate.php
│   │   └── Lane
│   │       ├── MaintenanceValueObject.php
│   │       └── PinsetterMaintenanceEntity.php
│   └── Repositories
│       ├── Janitor
│       │   ├── JanitorPaidSickLeaveScheduleRepository.php
│       │   └── JanitorWorkingHoursRepository.php
│       └── Lane
│           ├── LaneRepository.php
│           └── PinsetterMaintenanceScheduleRepository.php
├── Infrastructure
│   └── Repositories
│       ├── Janitor
│       │   ├── JanitorPaidSickLeaveScheduleEloquentRepository.php
│       │   └── JanitorWorkingScheduleEloquentRepository.php
│       └── Lane
│           ├── LaneRepositoryEloquentRepository.php
│           └── PinsetterMaintenanceScheduleEloquentRepository.php
└── UI
    ├── Api
    │   └── Controllers
    │       ├── Janitor
    │       │   ├── CafeteriaCoffeeTableCleaningController.php
    │       │   └── PinsetterMaintenanceController.php
    │       ├── Player
    │       │   ├── BowlingBallController.php
    │       │   ├── LaneController.php
    │       │   └── Scores
    │       │       ├── ChampionshipScoresController.php
    │       │       └── PracticeGameScoresController.php
    │       └── Rental
    │           ├── BowlingBallRentalController.php
    │           └── ShoesRentalController.php
    ├── Cli
    │   └── Commands
    │       └── Janitor
    │           └── PinsetterMaintenanceCommand.php
    └── Web
        ├── Controllers
        │   └── Janitor
        │       └── PinsetterMaintenanceController.php
        └── Views
            └── Janitor
                └── PinsetterMaintenanceView.blade.php
{% endhighlight %}

I hope that this was informative and that it helped you understand how to think about your code and how to better structure and abstract it into different layers.

This repository started out as an exercise for myself to help myself better understand DDD and how to incorporate its concepts into a Laravel app. But, I really liked how it turned out so I wanted to share it with this article, I also uploaded the full repository to GitHub so click the link below if you would like to check it out.

Have a nice one, see you in the next blog!

The full repository is located here:
- [https://github.com/oliverlundquist/laravel-ddd-bowling-alley](https://github.com/oliverlundquist/laravel-ddd-bowling-alley).
