---
layout:           post
title:            "Karma with Chrome Headless is not working in CI environment."
date:             2017-09-19 20:10:00 +0200
last_modified_at: 2017-09-21 18:58:00 +0200
tags:             [karma, chrome, headless, ci, circleci, selenium, docker, laravel, vue]
introduction:     "I stumbled upon a weird error in CircleCI 2.0 where I couldn't get Google Chrome to work in headless mode when running my VueJS tests in my Laravel app. After a couple of hours of trial and error I finally figured it out, I hope this will save some time for someone out there trying to achieve the same thing."
---

So I bumped into this error when testing my VueJS tests in my Laravel application in CircleCI 2.0 with the selenium/standalone-chrome:latest docker image.

{% highlight plaintext %}
ERROR [launcher]: Cannot start Chrome
[0919/104356.379812:ERROR:devtools_http_handler.cc(786)]
DevTools listening on 0.0.0.0:9222
[0919/104357.028892:ERROR:headless_shell.cc(132)] Navigation to  failed
{% endhighlight %}

After googling to no avail I decided that I'd get my hands dirty and started to dig to find a solution. I found that when installing Google Chrome Stable on Ubuntu it creates a bunch of symlinks that all trace back to this file:

{% highlight plaintext %}
lrwxrwxrwx 1 root root 31 Sep 19 04:51 /usr/bin/google-chrome -> /etc/alternatives/google-chrome
lrwxrwxrwx 1 root root 29 Sep 19 04:51 /etc/alternatives/google-chrome -> /usr/bin/google-chrome-stable
lrwxrwxrwx 1 root root 32 Sep 14 01:52 /usr/bin/google-chrome-stable -> /opt/google/chrome/google-chrome
-rwxrwxr-x 1 root root 3.0K Sep 19 12:24 /opt/google/chrome/google-chrome
{% endhighlight %}

Looking into this **/opt/google/chrome/google-chrome** file, the last lines where:

{% highlight plaintext %}
# Note: exec -a below is a bashism.
# DOCKER SELENIUM NOTE: Strait copy of script installed by Chrome with the exception of adding
# the --no-sandbox flag here.
#exec -a "$0" "$HERE/chrome" --no-sandbox "$PROFILE_DIRECTORY_FLAG" \
#  "$@"
#exec -a "$0" /etc/alternatives/google-chrome --no-sandbox "$@"
{% endhighlight %}

Looking at the error message my first thought was that the URL was not passed properly as an argument to the google-chrome binary, so I tried running the binary directly instead, which I found was located in **/opt/google/chrome/chrome** after a global find in the docker container.

I was pleasantly surprised that Google Chrome Stable ran just fine in headless mode when running it directly instead of through the binstub. So I ended up adding a command that tweaks the google-chrome binstub to run the binary directly instead.

This is the final **.circleci/config.yml** file:
{% highlight yaml %}
version: 2
jobs:
  karma:
    docker:
      - image: selenium/standalone-chrome:latest
    steps:
      - checkout
      - run: |
          head -n -6 /opt/google/chrome/google-chrome | sudo tee /opt/google/chrome/google-chrome-updated
          echo 'exec -a "$0" /opt/google/chrome/chrome --no-sandbox "$@"' | sudo tee -a /opt/google/chrome/google-chrome-updated
          sudo mv /opt/google/chrome/google-chrome-updated /opt/google/chrome/google-chrome
          sudo chmod 775 /opt/google/chrome/google-chrome
          ls -lah /usr/bin/google-chrome && ls -lah /etc/alternatives/google-chrome && ls -lah /usr/bin/google-chrome-stable && ls -lah /opt/google/chrome/google-chrome
      - run: |
          wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.4/install.sh | bash
          echo 'export NVM_DIR="$HOME/.nvm"' >> $BASH_ENV
          echo '[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"' >> $BASH_ENV
          echo '[ -s "$NVM_DIR/bash_completion" ] && \. "$NVM_DIR/bash_completion"' >> $BASH_ENV
      - run: nvm install 6.11.3
      - run: npm install
      - run: ./node_modules/karma/bin/karma start karma.config.js

workflows:
  version: 2
  karma_test_workflow:
    jobs:
      - karma
{% endhighlight %}

Hope this saves someone out there a couple of hours of debugging.
