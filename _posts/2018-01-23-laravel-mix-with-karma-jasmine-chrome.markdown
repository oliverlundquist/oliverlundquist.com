---
layout:           post
title:            "Vue Unit Testing in Karma with Laravel Mix using Jasmine, Webpack and Chrome."
date:             2018-01-23 09:58:00 +0700
last_modified_at: 2018-01-23 11:11:00 +0700
tags:             [laravel, mix, vue, javascript, js, unit, test, unit testing, karma, jasmine, webpack, chrome]
introduction:     "Use Laravel Mix's own Webpack config to setup unit testing with your Vue components and start testing your JS code to gain confidence by building solid apps with no bugs that actually work."
---

I am kind of a test junky and always try to get my apps code coverage close to 100%. It's not that I like writing tests which I do not think that anybody does, it's because I've had too many bad experiences with code related bugs.

There is nothing more frustrating than having to go back to old code and tweak it because it had bugs or did not work as expected. Not to mention when you tweak the code in one place that seemingly should not affect other parts actually creates more new bugs than what the actual bugfix solved.

In a recent Laravel project, I worked on I used Laravel Dusk to test the app thoroughly. I figured that since it tests your app in the browser with all assets loaded it should be as close to a real-world scenario as possible and should be a solid way of testing your app, right?

However, as much as I like Laravel Dusk I found E2E testing is too heavy to rely on solely. It takes too long time to launch the browser and load all assets and render the page before you can actually start testing your code I have switched to using E2E testing more as something that compliments other types of testing rather than the only and primary way.

I have found that unit testing JS code is really fast and if used with Karma Runner you can target different browsers and actually make the test run inside a real browser, which together with some E2E testing, to me feels like the most efficient way to test your app.

Now since you already spent time setting up Laravel Mix and you're happy with the results, I am going to use that Webpack config file in this setup process, that should also give you the most accurate results in terms of compiling your JavaScript code since that is the way you probably compile it for your other environments.

So without further ado, let's start by creating a new Laravel project.
{% highlight plaintext %}
composer create-project --prefer-dist laravel/laravel laravel-karma-jasmine-chrome
{% endhighlight %}

Then create a config file for Karma Runner with the filename karma.conf.js in the root of your repository
{% highlight javascript %}
var webpackConf = require('./node_modules/laravel-mix/setup/webpack.config.js');
delete webpackConf.entry

module.exports = function(config) {
    config.set({
        basePath: './resources/assets/js/',
        frameworks: ['jasmine'],
        files: [
            { pattern: 'components/*.spec.js', watched: false }
        ],
        exclude: [],

        webpack: webpackConf,
        webpackMiddleware: {
            noInfo: true,
            stats: 'errors-only'
        },
        preprocessors: {
            'app.js': ['webpack'],
            'components/*.spec.js': ['webpack']
        },

        reporters: ['progress'],
        port: 9876,
        colors: true,
        logLevel: config.LOG_INFO,
        autoWatch: true,
        browsers: ['ChromeHeadless'],
        singleRun: false,
        concurrency: Infinity
    })
}
{% endhighlight %}

Next, we need to install Karma Runner and other necessary NPM dependencies, like so:
{% highlight plaintext %}
npm install --save-dev karma karma-webpack karma-chrome-launcher karma-jasmine jasmine-core
{% endhighlight %}

We also need to install Laravel Mix and other Laravel NPM dependencies by running the npm install command:
{% highlight plaintext %}
npm install
{% endhighlight %}

Next, let's create a sample test file to see that things are working as expected, add the following contents to a file on this path, resources/assets/js/components/ExampleComponent.spec.js, this will test the ExampleComponent that is shipped with Laravel.
{% highlight javascript %}
import Vue from 'vue'
import ExampleComponent from './ExampleComponent.vue';

function renderComponent(Component, props) {
    const Ctor = Vue.extend(Component)
    const vm = new Ctor(props).$mount()
    return vm;
}

describe('ExampleComponent', () => {
    it('should have mounted method', function () {
        expect(typeof ExampleComponent.mounted).toBe('function')
    });
    it('should have right template', () => {
        const vm = renderComponent(ExampleComponent, {})
        expect(vm.$el.textContent).toContain('Example Component');
    });
});
{% endhighlight %}

If you have installed Karma globally you can now run karma start to run your tests, if you installed karma locally as proposed in this setup then add a new command to your package.json that looks like this:
{% highlight plaintext %}
"karma-start": "karma start"
{% endhighlight %}

Now you can run npm run karma-start to start the Karma Runner and execute your tests, try it out and you should see *SUCCESS!* written in beautiful green letters.

Congratulations! We now have a JavaScript unit testing environment up and running for your Laravel app. I hope that this article was helpful and that it will help someone out there looking for a way to set up their testing environment in a similar way to what was introduced in this article.

The full repository is located here:
- [https://github.com/oliverlundquist/laravel-karma-jasmine-chrome](https://github.com/oliverlundquist/laravel-karma-jasmine-chrome).
