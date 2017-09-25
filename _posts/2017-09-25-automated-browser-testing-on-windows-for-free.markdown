---
layout:           post
title:            "Automated browser testing on Microsoft Edge, Firefox and Chrome for free."
date:             2017-09-25 19:42:00 +0200
last_modified_at: 2017-09-25 19:42:00 +0200
tags:             [laravel, dusk, e2e, browser, test, windows, edge, chrome, firefox, circleci, ci]
introduction:     "Automate your Laravel 5.5 and Laravel Dusk 2.0 browser tests for free in Microsoft Edge, Firefox and Chrome. CircleCI 2.0 offers 4 containers for open-source projects without any cost so you can run all your browser tests in parallel which is really awesome. It also has support for storing artifacts so you can take screenshots in the browsers while testing to view different states of your app."
---

So I've been working on a frontend project recently in Laravel 5.5 and VueJS and I wanted to write browser tests to make sure that everything is working properly. When you are writing a frontend app and have clients accessing it from different environments and devices you have to make sure that it will work in different browsers. When the project is small and quick to test some might feel that it is easier and quicker to just start the browser and click away to test the application. However, once the project hits a certain size, you will surely realize that it is an impossible task for a human to test the whole application manually by simply clicking around to verify that things are working as they are supposed to. Even worse, picture the scenario when you have to do it in multiple browsers for each little small tweak you do. It is time to automate your browser tests and let the computer do the heavy lifting.

I strongly recommend CircleCI which is a great continuous integration service and a perfect place to get started with automating your browser tests in the cloud. CircleCI 2.0 has native docker support which is great because you can easily start containers with different browsers installed and run your tests easily. However, this does not work well with the Windows platform. After some researching, I was happy to come across this announcement [*[link]*](https://www.browserstack.com/test-on-microsoft-edge-browser#selenium-cloud) from another service that I love, BrowserStack, which said that they offer free automated testing for Microsoft Edge. When you sign up as a new user you also get 100 minutes of automated testing for free on any platform which you could use for some sample testing in Internet Explorer for example.

Let's set up Laravel 5.5 with Laravel Dusk 2.0 and start testing your app with browser tests in parallel on CircleCI. If you haven't got an account at [CircleCI](https://circleci.com/) or [BrowserStack](https://www.browserstack.com/) yet, head over there for a quick signup before proceeding with the following steps.

First, let's get a clean copy of Laravel 5.5 and install Laravel Dusk 2.0:
{% highlight plaintext %}
composer create-project --prefer-dist laravel/laravel laravel-browser-testing
cd laravel-browser-testing
composer require --dev laravel/dusk
composer require --dev browserstack/browserstack-local
php artisan dusk:install
{% endhighlight %}

After running these commands you should have your local environment up and running. Now we just have add a BrowserStack helper script since that is where your Microsoft Edge tests will run and a CircleCI config file to run the tests in parallel in Edge, Chrome and Firefox on CircleCI.

Create a Traits folder inside your tests folder and copy this file into that folder, tests/Traits/RunsBrowserStackLocal.php.
{% highlight php %}
<?php declare(strict_types=1);

namespace Tests\Traits;

trait RunsBrowserStackLocal
{
    /**
     * @var Process
     */
    protected static $bs_local;

    /**
     * Start BrowserStack Local
     */
    public static function startBrowserStackLocal()
    {
        $arguments = ['key' => env('BROWSERSTACK_ACCESS_KEY')];

        if (static::$bs_local === null) {
            static::$bs_local = new \BrowserStack\Local();
            static::$bs_local->start($arguments);
        }

        static::afterClass(function () {
            if (static::$bs_local !== null) {
                static::$bs_local->stop();
            }
        });
    }
}
{% endhighlight %}

After that, create a .circleci in the root of your repository and copy this file: .circleci/config.yml
{% highlight yaml %}
{% raw %}
php_environment: &php_environment
  environment:
    APP_URL: http://0.0.0.0:9000
    APP_ENV: testing
    APP_DEBUG: true
    APP_KEY: base64:3tsYILu9CPazfThXovQw+q7QeCsBvJ71yoJc9pgfrrM=

version: 2
jobs:
  build:
    docker:
      - image: oliverlundquist/php7:latest
        <<: *php_environment
    steps:
      - checkout
      - run: curl -sS https://getcomposer.org/installer | php
      - run: mv composer.phar /usr/local/bin/composer
      - run: composer self-update
      - restore_cache:
          keys:
            - composer-v1-{{ checksum "composer.json" }}
      - run: composer install -o -n --prefer-dist
      - save_cache:
          key: composer-v1-{{ checksum "composer.json" }}
          paths:
            - vendor
      - persist_to_workspace:
          root: /usr/local/bin
          paths:
            - composer

  chrome:
    docker:
      - image: oliverlundquist/php7:latest
        <<: *php_environment
      - image: selenium/standalone-chrome:latest
    steps:
      - checkout
      - restore_cache:
          keys:
            - composer-v1-{{ checksum "composer.json" }}
      - run:
          command: php artisan serve --host=0.0.0.0 --port=9000
          background: true
      - run: TEST_BROWSER=chrome php artisan dusk
      - store_artifacts:
          path: tests/Browser/screenshots
      - store_artifacts:
          path: storage/logs

  firefox:
    docker:
      - image: oliverlundquist/php7:latest
        <<: *php_environment
      - image: selenium/standalone-firefox:latest
        environment:
          SE_OPTS: -enablePassThrough false
    steps:
      - checkout
      - restore_cache:
          keys:
            - composer-v1-{{ checksum "composer.json" }}
      - run:
          command: php artisan serve --host=0.0.0.0 --port=9000
          background: true
      - run: TEST_BROWSER=firefox php artisan dusk
      - store_artifacts:
          path: tests/Browser/screenshots
      - store_artifacts:
          path: storage/logs

  edge:
    docker:
      - image: oliverlundquist/php7:latest
        <<: *php_environment
    steps:
      - checkout
      - restore_cache:
          keys:
            - composer-v1-{{ checksum "composer.json" }}
      - run:
          command: php artisan serve --host=0.0.0.0 --port=9000
          background: true
      - run: TEST_BROWSER=edge php artisan dusk
      - store_artifacts:
          path: tests/Browser/screenshots
      - store_artifacts:
          path: storage/logs

workflows:
  version: 2
  build_test_deploy:
    jobs:
      - build
      - chrome:
          requires:
            - build
      - firefox:
          requires:
            - build
      - edge:
          requires:
            - build
{% endraw %}
{% endhighlight %}

And lastly, tweak your DuskTestCase.php so that is looks like this file: tests/DuskTestCase.php
{% highlight php %}
<?php declare(strict_types=1);

namespace Tests;

use Laravel\Dusk\TestCase as BaseTestCase;
use Facebook\WebDriver\Chrome\ChromeOptions;
use Facebook\WebDriver\Remote\RemoteWebDriver;
use Facebook\WebDriver\Remote\DesiredCapabilities;
use Tests\Traits\RunsBrowserStackLocal;

abstract class DuskTestCase extends BaseTestCase
{
    use CreatesApplication, RunsBrowserStackLocal;

    /**
     * Prepare for Dusk test execution.
     *
     * @beforeClass
     * @return void
     */
    public static function prepare()
    {
        // static::startChromeDriver();
    }

    /**
     * Create the RemoteWebDriver instance.
     *
     * @return \Facebook\WebDriver\Remote\RemoteWebDriver
     */
    protected function driver()
    {
        // edge
        if (env('TEST_BROWSER') === 'edge') {
            $url          = 'https://' . env('BROWSERSTACK_USERNAME') . ':' . env('BROWSERSTACK_ACCESS_KEY') . '@' . env('BROWSERSTACK_SERVER') . '/wd/hub';
            $capabilities = $this->browserStackCaps();

            static::startBrowserStackLocal();
            return RemoteWebDriver::create($url, $capabilities);
        }

        // Firefox
        if (env('CIRCLECI') === true && env('TEST_BROWSER') === 'firefox') {
            $url          = 'http://0.0.0.0:4444/wd/hub';
            $capabilities = DesiredCapabilities::firefox();

            return RemoteWebDriver::create($url, $capabilities);
        }

        // Chrome
        if (env('CIRCLECI') === true && env('TEST_BROWSER') === 'chrome') {
            $url          = 'http://0.0.0.0:4444/wd/hub';
            $capabilities = DesiredCapabilities::chrome();

            return RemoteWebDriver::create($url, $capabilities);
        }

        // Chrome (local)
        $options = (new ChromeOptions)->addArguments([
            '--disable-gpu',
            '--headless'
        ]);

        return RemoteWebDriver::create(
            'http://localhost:9515', DesiredCapabilities::chrome()->setCapability(
                ChromeOptions::CAPABILITY, $options
            )
        );
    }

    /**
     * Setup the BrowserStack capabilities
     *
     * @see https://www.browserstack.com/automate/capabilities
     * @return array
     */
    private function browserStackCaps()
    {
        $caps = [
            'project'              => config('app.name'),
            'browserstack.local'   => true,
            'browserstack.console' => 'info',
            // 'browserstack.debug'   => true,
            'os'                   => 'Windows',
            'os_version'           => '10',
            'browser'              => 'Edge',
            'browser_version'      => '15',
            'resolution'           => '1024x768'
        ];

        return $caps;
    }

    /**
     * Disable storeConsoleLogsFor for BrowerStack because getLogs seems to be unsupported by many browsers
     *
     * @param  \Illuminate\Support\Collection $browsers
     * @return void
     */
    protected function storeConsoleLogsFor($browsers)
    {
        return;
    }
}
{% endhighlight %}

Now your repository is all set, so go ahead and create a new repository on GitHub and push the files. Now head over to BrowserStack and copy your username and access key, we need to add these to the CircleCI environment to be able to access BrowserStack from CircleCi and run the browser tests on Microsoft Edge. Next step is to log in to CircleCI, click "add project", select CircleCI 2.0 and click start building to add your new GitHub repository. Then add the following environment variables in the environments variables menu to the left on your project settings page.

{% highlight plaintext %}
BROWSERSTACK_SERVER=hub-cloud.browserstack.com
BROWSERSTACK_USERNAME=<YOUR USERNAME>
BROWSERSTACK_ACCESS_KEY=<YOUR ACCESS_KEY>
{% endhighlight %}

Click rebuild or push a new commit to your GitHub repository to trigger a build. All application logs and artifacts will be stored and linked to in the artifacts tab in each build details page.

You can get the full repository [here](https://github.com/oliverlundquist/laravel-browser-testing) to check for differences if you are encountering any issues.

Hope this helps someone out there!
