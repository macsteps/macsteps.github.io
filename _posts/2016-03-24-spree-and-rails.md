---
layout: post
title:  "Spree and Rails"
date:   2016-03-24 16:20:27 -0700
categories: spree rails
---
# Spree and Rails

In this post, I outline the steps taken to setup a new Rails site using the [Spree E-commerce Engine](https://github.com/spree/spree). You can find instruction in their README.md or their [Guides](https://github.com/spree/spree/pull/7199).

Note that I wanted a clean install and did not load their sample data. While I did that the first time, I wanted to start a new site without their sample content.

## Install

Create a new Rails app. I use PostgreSQL in development, so I can work with it and understand it for times when I need to troubleshoot in other environments (e.g. test, production).

```
# omit the '-d postgresql' if you want to use default SQLite3.
rails new myapp -d postgresql
cd myapp
```

Add Spree to the Gemfile:

```
gem 'spree',                    '~> 3.0', '>= 3.0.7'
gem 'spree_auth_devise',        '~> 3.0.5'
gem 'spree_gateway',            '~> 3.0.0'
gem 'deface',                   '~> 1.0', '>= 1.0.2'
```

A few notes here:
+ For versions, I recommend you go to [Ruby Gems](https://rubygems.org/) and just grab the latest.
+ I've added [Deface](https://github.com/spree/deface), which you might as well do now, because that's how you make changes to views using this Engine.
+ Spree is built to work with [Devise](https://github.com/plataformatec/devise) for authentication, which is well-respected.
+ [Spree Gateway](https://github.com/spree/spree_gateway) provides the code (as a wrapper for Active Merchant) for working with payment gateways.

Let's finish the installation:

```
bundle install                # install spree gems
# omit the arguments on the next line if you want a populated sample site.
rails g spree:install --sample=false --seed=false
rails g spree:auth:install    # integration with devise
rails g spree_gateway:install (y to migrations)    # payment gateway code
rake spree_auth:admin:create  # create an admin user
rails s                       # start the webrick server
```

### Verify the install went well

+ Visit http://localhost:3000 to see the home page.

+ Visit http://localhost:3000/admin
  - Log in with credentials specified during the admin creation.
  - Add a product, if you want to do that now (it will make the currently desolate site more interesting, if only slightly.)

## Customize

Now I'll describe a few changes that I made initially.

### Create css files local to the app by copying from gem dir

```
cp <path_to_ruby>/gems/spree_frontend-3.0.8/app/assets/stylesheets/spree/frontend/frontend_bootstrap.css.scss ~/workspace/myapp/vendor/assets/stylesheets/spree/frontend/
cp <path_to_ruby>/gems/spree_frontend-3.0.8/app/assets/stylesheets/spree/frontend/_variables.scss ~/workspace/myapp/vendor/assets/stylesheets/spree/frontend/
```

> <path_to_ruby> depends on your system and how ruby is installed.
>> For example, I use RVM on a Mac, so <path_to_ruby> = ~/.rvm/gems/ruby-2.2.1/

Once copied, I simply edited the copy of frontend_bootstrap.css.scss to change:

```
#spree-header {
  background: $brand-primary asset-url('spree_header.jpg') center center;
  background-size: cover;
  margin-bottom: $line-height-computed;
```

to:

```
#spree-header {
  background: none;
  background-color: #996633;
  margin-bottom: $line-height-computed;
```

### change logo file
```
mkdir -p vendor/assets/images/logo
cp logo_file myapp/vendor/assets/images/logo/spree_50.png
```

### remove All Departments dropdown from header

My site wasn't going to be that large initially, so this looked like overkill. It's also a good example of what needs to be done to change views.

In the myapp directory:
```
mkdir -p app/overrides/spree/shared
```

Next, create overrides nav_overrides.rb to remove dropdown from \_search.html.erb.

If you're asking yourself how I knew to edit this file, I did so by reading the code. That's mostly what you'll be doing, at least initially, if you want to work with Spree.

There are two navigation bars on the page. In this case, the one with the search is called \_nav_bar.html.erb. In there you will see this bit of code:

```
<%= render :partial => 'spree/shared/search' %>
```

Looking at \_search.html.erb, I see the code I want to remove:

```
<div class="form-group">
  <% cache(cache_key_for_taxons) do %>
    <%= select_tag :taxon,
          options_for_select([[Spree.t(:all_departments), '']] +
                                @taxons.map {|t| [t.name, t.id]},
                                @taxon ? @taxon.id : params[:taxon]), 'aria-label' => 'Taxon', class: "form-control" %>
  <% end %>
</div>
```

The code '<div class="form-group"' appears elsewhere, so I'll have to start with the cache line. Using [Deface](https://github.com/spree/deface), the code I need will go into myapp/app/overrides/spree/shared/nav_overrides.rb:

```
Deface::Override.new(:virtual_path => "spree/shared/_search",
                      :name => "search",
                      :remove => "erb[silent]:contains('cache(cache_key_for_taxons) do')",
                      :closing_selector => "erb[silent]:contains('end')")
```

The [README.md](https://github.com/spree/deface) does a good job of explaining options as does the [Spree Deface Guide](http://guides.spreecommerce.com/developer/view.html).

+ :virtual_path is the relative location of that file.
  - for example: I use [RVM](https://rvm.io/) on a Mac and my location is ~/.rvm/gems/ruby-2.2.1/gems/spree_frontend-3.0.8/app/views/spree/shared
+ :name is arbitrary, but should make sense to you. This may prove useful in logs.
+ :remove is the command and there are several.
+ :closing_selector is used, so that Deface knows when to stop removing code.

### Make sidebar appear on home page

The home page is definitely lacking without the sample data. Since I added a product earlier, I wanted to make the sidebar appear.

+ In a browser, http://localhost:3000/admin/taxonomies
+ Add a Taxonomy, which can be thought of as a broad category. For example, Tea.
+ Add a Taxon (or two), which can be thought of as a sub-category. For example, Dragonwell, Sencha, or Gyokuro.


### Conclusion

I need to add products and work on the data entry aspect of getting product descriptions, shipping data, and tax data into the system. Looks like I'll need some of that tea for this type of work.
