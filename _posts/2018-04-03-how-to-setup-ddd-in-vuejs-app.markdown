---
layout:           post
title:            "How to setup Domain Driven Design (DDD) in a Vue.js app."
date:             2018-04-04 14:25:10 +0900
last_modified_at: 2018-04-04 15:37:20 +0900
tags:             [laravel, vue, vuejs, vue.js, ioc, container, dependency, injection, di, ddd, design, structure, layers, ui, application, domain, infrastructure]
introduction:     "In this article, I will demonstrate how you can create a Vue.js app with the DDD approach in TypeScript that has dependency injection and logic clearly separated into the 4 different DDD layers."
---

I have been working a lot in Laravel and Vue.js environments recently when developing apps. Some of these apps are getting big and the folder structure easily gets awkward and poorly structured once the app grows.

Ever since I started incorporating Domain Driven Design concepts seriously in my projects a couple of weeks ago, it has dominated the way I structure and think about code. I even wrote an article a couple of weeks ago explaining how to structure a Laravel app in a DDD manner and I totally loved the outcome. If you haven't read it or if you are not really clear about what DDD is yet, head over to the [article](/2018/03/20/how-to-setup-ddd-in-laravel-app.html) first to readup on the topic because it explains a lot of the core concepts and layer structured design patterns in DDD that I will not explain in this article.

### IoC, DI and many things that need to be solved

So, if I could do it in a Laravel app as explained in the previous article, why not try to do it in a Vue.js app, right? Well, there are many things that are missing in Vue.js out of the box to make this setup smooth and pain-free. That's why I felt that I needed to write another blog article about DDD and how to incorporate it in a Vue.js project.

- First, we need an IoC (Inversion of Control) container so that we can abstract our repositories in our domain layer.

- Secondly, we need to figure out how to DI (Dependency Injection) TypeScript classes into Vue.js components to make the Vue.js logic as compact as possible.

After a lot of trial and error, I believe I came up with a great solution for how to make a solid foundation for a Vue.js that scales with the application as it gets bigger.

### So how did I do it?

After looking for IoC container libraries out there I found the awesome [InversifyJS](http://inversify.io/){:target="_blank"} library for TypeScript so I decided to take it for a spin. Having no idea where to start or how to inject classes into Vue.js components I stumbled upon this [article](https://blog.kloud.com.au/2017/03/22/dependency-injection-in-vuejs-app-with-typescript/){:target="_blank"} which was really helpful and helped me get started.

After having just found the Punk API 2.0 (a public Brewdog Beers JSON API) a couple of days prior to when I wrote this article, I thought it would be a great API to use for a demo, because who doesn't like beer, right? But also, since it's common for apps to communicate with external APIs it would be useful to see how this code gets structured in the app.

### Let's build this thing!

![Brewdog Demo App Screenshot](/assets/brewdog-demo-app-screenshot.png){:class="image-with-border"}
*This is what we're building ladies and gentlemen.*

Ok, so first thing's first, we need to make a couple of modifications to a stock Laravel app to setup the TypeScript and Vue.js environment.

1: Let's create a Laravel project
{% highlight plaintext %}
composer create-project --prefer-dist laravel/laravel laravel-vue-ddd-brewdog
{% endhighlight %}

2: Let's create a TypeScript config file
{% highlight json %}
{
    "compilerOptions": {
        "target": "es5",
        "lib": ["es6", "dom"],
        "types": ["reflect-metadata"],
        "strict": true,
        "strictPropertyInitialization": false,
        "module": "es2015",
        "moduleResolution": "node",
        "experimentalDecorators": true,
        "emitDecoratorMetadata": true
    },
    "include": [
        "resources/assets/js/**/*"
    ]
}
{% endhighlight %}

3: Let's install all the NPM dependencies that we need for this demo
{% highlight plaintext %}
npm install && npm install --save-dev laravel-mix@^1.7.2 ts-loader@^3.5.0 typescript vue-class-component vue-router vue-i18n vuex vuex-class inversify reflect-metadata @types/lodash
{% endhighlight %}

4: Let's tweak Laravel's default view, resources/views/welcome.blade.php, to make it load Vue.js
{% highlight html %}
{% raw %}
<!doctype html>
<html lang="{{ app()->getLocale() }}">
    <head>
        <meta charset="utf-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width, initial-scale=1">
        <meta name="csrf-token" content="{{ csrf_token() }}">
        <link rel="stylesheet" href="{{ mix('/css/app.css') }}">
        <title>Vue.js Domain Driven Design app with Vuex, Vue-Router and TypeScript</title>
    </head>
    <body>
        <div id="app"></div>
        <script>
            window.laravel_config = {
                APP_ENV: "{{ env('APP_ENV') }}",
                locale: "{{ config('app.locale') }}",
                fallbackLocale: "{{ config('app.fallback_locale') }}"
            }
        </script>
        <script src="{{ mix('/js/bootstrap.js') }}"></script>
        <script src="{{ mix('/js/app.js') }}"></script>
    </body>
</html>
{% endraw %}
{% endhighlight %}

5: Let's add a wildcard route to routes/web.php to prevent Laravel from returning 404 when you're accessing the app directly with a path that is only inside the Vue Router
{% highlight php %}
Route::get('{view}', function () {
    return redirect('/');
})->where('view', '.*');
{% endhighlight %}

6: Let's tweak the Laravel Mix config located in the webpack.mix.js to make it compile TypeScript
{% highlight plaintext %}
mix
    .js('resources/assets/js/bootstrap.js', 'public/js')
    .ts('resources/assets/js/app.ts', 'public/js')
    .sass('resources/assets/sass/app.scss', 'public/css');
{% endhighlight %}

Now we're all done setting up the repository and we can start building the app! 🎉

### Getting down to business ... logic

Now we're all set and can start building the actual app. In this section, I'm not going to explain and paste every content from every file since there are so many. I will rather explain how I structured the code while you can check the content of the files yourself in the final repository that I've pushed to GitHub, located here: [oliverlundquist/laravel-vue-ddd-brewdog](https://github.com/oliverlundquist/laravel-vue-ddd-brewdog/tree/master/resources/assets/js){:target="_blank"}

If you open the apps/brewdog folder you will see all the four layers of the DDD approach separated into different folders, I will go through them one by one. As I explained in my earlier [article](/2018/03/20/how-to-setup-ddd-in-laravel-app.html) there are four layers in DDD, structured from top to bottom, UI, Application, Domain and Infrastructure that the application request flows through.

#### First up, UI

One of the biggest challenges was to confine the Vue.js logic into only one layer in the application and that is exactly what we are aiming to do in this layer. We are keeping Vue.js components and view presentation logic here while injecting arbitrary TypeScript classes from the application layer to keep things separate and clean. As you can see there are only files in the Web subfolder as for now but we could reuse the application logic into an API or a CLI tool since we are not coupling the UI with the rest of the application and business logic.

#### Next, Application!
This is where we tie the other two lower layers (domain and infrastructure) together to make the application work. Here you will find all the bindings for the IoC container and also other application services which are features that your application can respond to.

#### Domain layer!
This is where the business logic goes, preferably this layer should not have any dependencies on other layers and should be as decoupled and separate as possible. The files in the repositories folder are only interfaces and contracts that indicate what data the domain layer is using but not necessarily how and where it gets it from. That brings us to the next and last layer, the infrastructure layer.

#### Last but not least, the Infrastructure layer.
This is where all the implementation details for how data is stored in a database, how data is fetched from external APIs, how state is persisted and so on. In this demo, we're using Vuex and the Vue.js Web UI components use this for handling state, so that's why that is there.

That's it for this time, I hope this was informative and that it might have given you some inspiration on how you can structure your next Vue.js project.

Until next time, have a good one!

Don't forget to check out the full repository here:
- [https://github.com/oliverlundquist/laravel-vue-ddd-brewdog](https://github.com/oliverlundquist/laravel-vue-ddd-brewdog).
