---
layout: post
title:  "Heroku Production Configurations"
date:   2016-03-24 16:20:27 -0700
categories: deploy heroku production rails
---
# Heroku Production Configurations

I just completed setting up a couple of important configurations for a new application, and I run production on [Heroku](https://www.heroku.com), who offer excellent and reliable tools for managing Rails (among others) applications.

## Puma Webserver

[Heroku recommends](https://devcenter.heroku.com/articles/ruby-default-web-server) running Puma instead of WEBrick for production, primarily due to the latter being single-threaded and the former being multi-threaded. The bottom line is that configuring Puma for production will prepare the application for moderate to heavy usage.

I recommend reading [Heroku's deploying with Puma doc](https://devcenter.heroku.com/articles/deploying-rails-applications-with-the-puma-web-server). In short, I needed two files.

Create ```Procfile``` at the Rails root:

```
web: bundle exec puma -C config/puma.rb
```

And create ```config/puma.rb```:

```
workers Integer(ENV['WEB_CONCURRENCY'] || 2)
threads_count = Integer(ENV['MAX_THREADS'] || 5)
threads threads_count, threads_count

preload_app!

rackup      DefaultRackup
port        ENV['PORT']     || 3000
environment ENV['RACK_ENV'] || 'development'

on_worker_boot do
  # Worker specific setup for Rails 4.1+
  # See: https://devcenter.heroku.com/articles/
  # deploying-rails-applications-with-the-puma-web-server#on-worker-boot
  ActiveRecord::Base.establish_connection
end
```

Both configurations I learned from [Michael Hartl's Ruby Tutorial](https://www.railstutorial.org/).

Send both to the git repo (e.g. Github) and Heroku:

```
git add Procfile
git add config.puma.rb
git commit -m "configure production to use Puma on Heroku."
git push
git push heroku master
```

Verify with ```heroku logs``` and you should see something like:

```
2016-03-26T22:28:49.849615+00:00 app[web.1]: [3] Puma starting in cluster mode...
2016-03-26T22:28:49.849632+00:00 app[web.1]: [3] * Version 3.2.0 (ruby 2.2.4-p230), codename: Sprin
g Is A Heliocentric Viewpoint
2016-03-26T22:28:49.849634+00:00 app[web.1]: [3] * Min threads: 5, max threads: 5
2016-03-26T22:28:49.849642+00:00 app[web.1]: [3] * Process workers: 2
2016-03-26T22:28:49.849641+00:00 app[web.1]: [3] * Environment: production
```

## Sendgrid Email Server

I'm working with [Devise](https://github.com/plataformatec/devise) for authentication and decided to immediately start with ```:confirmable```, which uses a user registration process:

1. User signs up
2. An email is sent to the user
3. The user clicks the link
4. The server confirms the user
5. The user can now log in

The purpose of this process is to confirm that the user has provided a valid email address.

There's two ways to add Sendgrid to your app, though I've only used the first:

1. [Heroku Resources page](https://dashboard.heroku.com/apps/lnat/resources)
2. Run ```heroku addons:create sendgrid:starter``` from Rails root

I chose the former, because I also needed to add credit card information to my free account. Gotta have some accountability for the spammers and other "evildoers".

It happens immediately, so next modify ```config/environments/production.rb```:

```
config.action_mailer.raise_delivery_errors = true
config.action_mailer.default_url_options = { :host => "http://my-app-name.herokuapp.com" }
config.action_mailer.smtp_settings = {
  :address        => 'smtp.sendgrid.net',
  :port           => '587',
  :authentication => :plain,
  :user_name      => ENV['SENDGRID_USERNAME'],
  :password       => ENV['SENDGRID_PASSWORD'],
  :domain         => 'heroku.com',
  :enable_starttls_auto => true
}
```

Like many settings, the Sendgrid username and password are configured by Heroku as environment variables, so this file simply reads them. Storing this information in a Git repo is [poor security practice](https://www.owasp.org/index.php/OWASP_AppSec_DC_2012/Friends_dont_let_friends_store_passwords_in_source_code).

Checkout ```heroku config``` for a list of environment variables configured on your Heroku dynos. You can also query a specific variable with ```heroku config:get SENDGRID_USERNAME```, for example.

Commit and push:

```
git add config/environments.production.rb
git commit -m "configure production to use Sendgrid on Heroku."
git push
git push heroku master
```

I confirmed using my application with ```heroku logs --tail```:

```
2016-03-26T23:26:35.841021+00:00 app[web.1]: Sent mail to testuser@example.com (1737.8ms)
2016-03-26T23:26:35.841026+00:00 app[web.1]: Date: Sat, 26 Mar 2016 23:26:34 +0000
2016-03-26T23:26:35.841027+00:00 app[web.1]: From: from@example.com
2016-03-26T23:26:35.841028+00:00 app[web.1]: Reply-To: from@example.com
2016-03-26T23:26:35.841029+00:00 app[web.1]: To: testuser@example.com
```

Note that I changed the above output to mask the real email address I used.

## Compile Assets

I added a logo file to ```app/assets/images```, and while it showed up fine in development, it did not in production.

First, add [Rails 12 Factor](https://github.com/heroku/rails_12factor#readme), which provides for logging to STDOUT and serve static assets, to your ```Gemfile```:

```
gem 'rails_12factor', group: :production
```

Then run ```bundle install```.

Next, modify ```config/environments/production.rb```, which defaults to false:

```
config.assets.compile = true
```

Now instead of seeing:

```
ActionController::RoutingError (No route matches [GET] "/assets/logo.png"):
```

in your logs, you should see:

```
Started GET "/assets/logo.png"
```

## Conclusion

Getting setup on Heroku requires a few steps that are both relatively easy to understand and well-documented.
