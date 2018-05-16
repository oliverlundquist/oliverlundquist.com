---
layout:           post
title:            "How to setup code style linting for TypeScript with TSLint."
date:             2018-05-15 14:25:10 +0900
last_modified_at: 2018-05-15 15:37:20 +0900
tags:             [laravel, tslint, setup, howtosetup, styling, codestyling, lint, linting]
introduction:     "I've been writing a lot of TypeScript with Vue lately, being a big fan of php-cs-fixer and style linting I decided to setup a structure for style linting of my TypeScript code as well."
---

I'm a big fan of style linting and php-cs-fixer is one of the dependencies that I install first when starting on a new Laravel or PHP project. I've been working in various development teams and usually, developers have their own strong preference on how code should be styled. Code styling doesn't affect how the code will perform but still, it's something that is often argued about and something that is hard to reach a consensus about.

So, instead of spending time "bike-shedding" how to style code in your team, it's way more effective to just use a code style linting tool and stop arguing about code styling to get on developing and thinking about the more important issues at hand.

That said, I am quite familiar with setting up style linting in PHP projects, but haven't done so with TypeScript code before so I thought that I would write a quick run through on how I did it. As a base, I will use my Laravel and Vue with TypeScript repository that I wrote an article about earlier in this blog, you can find the article [here](/2018/04/04/how-to-setup-ddd-in-vuejs-app.html) and the repository [here](https://github.com/oliverlundquist/laravel-vue-ddd-brewdog){:target="_blank"}.

Start with installing TSLint to your repository like so:
{% highlight plaintext %}
npm install tslint --save-dev
{% endhighlight %}

If you have npx installed you do not have to do this but I find it useful to add commands in package.json, this is optional but if you want, add these two commands to your package.json file.
{% highlight plaintext %}
"tslint": "tslint 'resources/assets/**/*.ts?(x)' --exclude resources/assets/js/vue-i18n-locales.generated.ts --format verbose"
"tslint-fix": "tslint 'resources/assets/**/*.ts?(x)' --exclude resources/assets/js/vue-i18n-locales.generated.ts --format verbose --fix --force"
{% endhighlight %}

Lastly, add a config for TSLint by adding a file named `tslint.json` in the root of your repository. I like to use single-quotes when double-quotes are not necessary so I've added that extra rule, otherwise, I will always try to keep my setup as close as possible to the recommended defaults.
{% highlight json %}
{
    "defaultSeverity": "error",
    "extends": [
        "tslint:recommended"
    ],
    "jsRules": {},
    "rules": {
        "quotemark": [true, "single", "avoid-escape", "avoid-template"]
    },
    "rulesDirectory": []
}
{% endhighlight %}

Now simply run `npm run tslint` to check your repository for style linting errors and `npm run tslint-fix` to fix them. Checking your repository for style linting errors without fixing them can be useful in a CI environment. This will give you feedback on if new code that is about to get merged (in pull requests on GitHub etc.) is following the code styling standards that you've set up.

I hope this was useful and that it saved someone out there a couple of hours of trial and error setting something like this up.

Until next time, have a good one!
