---
layout: post
title: "AppSignal, Capistrano and environment variables"
description: ""
category: partheas
tags:
  - ruby
  - rails
  - monitoring
---
{% include JB/setup %}

For a [customer](http://www.stroomchecker.be/) of [Partheas](http://partheas.com/), I
had to foresee better monitoring for a Ruby on Rails application. After comparing
a few I set on trying out [AppSignal](https://appsignal.com/).

I haven't tested them enough to give a thorough review, but I did run into one
minor issue due to how we set up our staging environment and production environment.

For simplicity both staging and production run with a Rails environment set to
production and when a different setting is needed we use environment variables.
So I was happy to see that the AppSignal configuration file supported this out of
the box.

AppSignal uses deploy hooks "[..] to measure improvements in your app and as a
way to re-set exception notifications.", for this they provide a default Capistrano
integration which needs an API key being present on the machine you're deploying
from. To circumvent this need I removed the default Capistrano integration and
replaced it by a call to their cli that runs on the deployed host.

    after "deploy", "appsignal:deploy"
    after "deploy:migrations", "appsignal:deploy"

    namespace :appsignal do
      task :deploy do
        run "cd #{current_path} ; bundle exec appsignal notify_of_deploy " \
            "--revision=#{current_revision} " \
            "--repository=#{repository} " \
            "--user=#{ENV['USER']} " \
            "--environment=#{rails_env}"
      end
    end

I'll be posting a real review of it soon.
