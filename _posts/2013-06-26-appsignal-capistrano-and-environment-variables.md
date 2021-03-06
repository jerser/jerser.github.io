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

I needed monitoring for a Ruby on Rails application, after comparing
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

{% highlight ruby %}
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
{% endhighlight %}

And our config/appsignal.yml looks like this.

{% highlight yaml %}
---
development:
  active: false
production:
  api_key: "<%= ENV['APPSIGNAL_API_KEY'] %>"
  active: true
{% endhighlight %}

UPDATE 2014: [We](http://www.partheas.com) still use AppSignal for monitoring
our Rails applications. We're happy with it, so go use it yourself!
