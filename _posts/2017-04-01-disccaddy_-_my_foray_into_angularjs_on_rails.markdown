---
layout: post
title:  "DiscCaddy - My Foray Into AngularJS on Rails"
date:   2017-04-01 16:38:37 +0000
---


When setting out to build my final project for the Flatiron School, I knew I wanted to make something different. I actually started multiple projects, but didn't like where they were headed. I finally came up with an idea that resonated with me.

Disc golf is a relatively unknown sport, so there aren't a ton of resources out there for it. Each disc has it's own special characteristics, flight-patterns, weight, material, etc. Even as a newer player, it can be hard to keep track of which discs you have. I had a cluttered note file on my phone, filled with disc names, weights, and how they flew for me. It was a major eye-sore. This led me to create DiscCaddy.

The immediate goal of DiscCaddy is to have a comprehensive list of discs, sorted by type, and allow a user to create a virtual collection of their discs. Soon, I plan to add a lot more features like a course list, shareable lists, notes, friends, etc. For now, I just wanted to get the basic app up and running.

This post will assume you're familiar with the basics of Ruby on Rails, and AngularJS. 

## Part One - AngularJS on Rails

The first major obstacle was figuring out how to get AngularJS and Rails to play nicely together.

To do this, we'll use [Bower. ](https://bower.io/) Bower is a package manager - meaning it will find, download, and save the stuff you're looking for. (Note: you may have to install bower on your local machine, in which case follow the instructions in the provided link.)

Let's add the `bower-rails` gem to our Gemfile. While we're there, let's go ahead and add `angular-rails-templates`, which allows us to use Angular templates in our Rails asset pipeline. This removes the need for AJAX calls to retrieve the templates. Also, we'll add the `active-model-serializers` gem, which lets Rails serialize your chosen models into JSON for Angular to use.

`Gemfile.rb`

```
gem 'bower-rails'
gem 'angular-rails-templates'
gem 'active_model_serializers'
```

Then, in terminal, run:

```
bundle install
rails g bower_rails:initialize json
```

This will create a file called `bower.json` in the root directory of your app. This is where we'll tell Bower which packages we want. In this case, we want angular, and angular ui-router. At the very least, for this section, we'll want angular. (Note: I also added a dependency for angular-bootstrap and angular-devise in my application, so you may want to do that here as well.)

```
{
  "vendor": {
    "name": "bower-rails generated vendor assets",
    "dependencies": {
      "angular": "v1.5.8",
      "angular-ui-router": "latest"
    }
  }
}
```

In terminal, run `rails bower:install` to get everything set up.

This will create a folder in `vendor/assets` called `bower_components`. We don't necessarily want to push all these components up to github. Rather, a user could just run `rails bower:install` and set up the dependencies themselves. So in `.gitignore`, add the following:

```
# Ignore Bower components
/vendor/assets/bower_components
```

At this point, we need to add these new dependencies to our `application.js` file, and remove any references to turbolinks. Turbolinks and Angular/jQuery don't play well together.

`application.js`

```
//= require jquery
//= require angular
//= require angular-ui-router
//= require angular-rails-templates
//= require angular-devise
//= require bootstrap-sprockets 
//= require_tree .
```

We'll need to remove any references to turbolinks in `application.html.erb` as well, which should look like this when completed:

```
<%= stylesheet_link_tag    'application', media: 'all' %>
<%= javascript_include_tag 'application' %>
```

At this point, you should be able to run your rails server and see the beautiful rails welcome page.

That's great and all, but we came here to see Angular! Don't fret, it's coming.

First let's define a root route in `config/routes.rb`:

```
root 'application#index'
```

Then create the appropriate method in `controllers/application_controller.rb`:

```
def index
end
```

And the view in `views/application/index.html.erb`:

```
<ui-view></ui-view>
```

This is where our Angular app will get loaded.

Speaking of our Angular app... let's create it!

Under `app/assets/javascripts` create a file called `app.js`. That file will look something like this:

```
(function () {
  'use strict';
  
  angular
    .module('app', ['ui.router', 'templates'])

}());
```

Here we've defined our app, `'app'`, and our dependencies, `'ui.router', 'templates'`. 

In order for this Angular app to show up in our Rails app, we'll pick an HTML element in `application.html.erb` and add `ng-app="app"`.

```
<html ng-app="app">
...
```

Now let's create an angular route. In `app/assets/javascripts` create a file called `routes.js`:

```
(function () {
  'use strict';

  angular
    .module('app')
    .config(function($stateProvider, $urlRouterProvider) {
      $stateProvider
        .state('home', {
          url: '/',
          templateUrl: 'home/home.html',
          controller: 'HomeController as vm'
        })
    })

}());
```

Here we use `$stateProvider` to define our route, 'home', which has a template file and a controller we'll need to create.

Create a folder in `app/assets/javascripts` called `home`. In that folder, create `home.html` and `home.controller.js`.

Our simple `HomeController` looks like this:

```
(function () {
  'use strict';

  angular
    .module('app')
    .controller('HomeController', HomeController)

    function HomeController() {
      var vm = this

      vm.compliment = "you're awesome."
    }  

}());
```

And our `home` template looks like this:

```
<h1>Hello, {{ vm.compliment }}</h1>
```

Voila! Angular application communicating with our Rails application!

## Part Two

Now I'm going to skip ahead a bit, and get back to DiscCaddy.

DiscCaddy allows users to create a virtual collection of discs they have. So we have a `User` model in rails. A `User` will have many discs, through a `Bag` model. The `Disc` model will have many users, through the `Bag` model. We'll focus on the `Disc` model.

### The `Disc` model

The meat and potatoes of this app are the discs.

```
create_table "discs", force: :cascade do |t|
    t.string       "name"
    t.text         "description"
    t.datetime     "created_at",      null: false
    t.datetime     "updated_at",      null: false
    t.string       "disc_type"
    t.integer      "low_weight"
    t.integer      "high_weight"
    t.string       "thumbnail_url"
    t.string       "image_url"
    t.integer      "selected_weight"
  end
```

There are a lot of different disc options, and I wanted to quickly create a bunch without having to repeatedly assign values to all those properties manually.

#### Building a scraping task with Nokogiri

I decided to make a rake task to scrape a certain website for information on discs. To do this (for whatever project you're working on), in your `lib/tasks` folder, create a `.rake` file. I named mine `disc_scraper.rake`, you choose whatever name you like.

```
namespace :scraper do
  require 'nokogiri'
  require 'open-uri'
end
```

I start by giving it a `namespace` of `:scraper`. This allows me to call `rake scraper:task_name` in the terminal to run the task. In this case I'm just going to call the task `scrape`, and give it a brief description.

```
namespace :scraper do
  require 'nokogiri'
  require 'open-uri'
	
  desc 'Scrape all discs'
  task scrape: :environment do

  end
end
```

Now, in the terminal, if you run `rake scraper:scrape` it will run the code you put in the task block.

I started out scraping just one disc type until I found the right css selectors. Then the challenge became - how do I keep it DRY but still scrape 4 (or more) different URLs? Well, I made a `urls` hash, and will loop through it with `.each` later on.

I'm going to keep this section a little generic just to show how I went about building up the discs. The selectors will be different for every project.

```
urls = {
  'type1': 'http://type1.com',
  'type2': 'http://type2.com',
  'type3': 'http://type3.com',
  'type4': 'http://type4.com
}
```

Inside the `scrape` block, I made a method called `scrape_discs`. This method will take in two arguments, `type` and `url`. These correlate with the format of the `urls` hash.

So in the scrape method we need to do a few things:

1. Use Nokogiri and open-uri to scrape the URL and assign it to a variable, `doc`
2. Find the CSS selector to grab the objects you want
3. Iterate through the collection of objects and find the individual CSS selectors for the desired properties

```
def scrape_discs(disc_type, url)
  doc = Nokogiri::HTML(open(url))
	
  discs = doc.css('.selector')
	
  discs.each do |disc|
    name = disc.css('.selector')
    type = disc_type
    link = disc.css('.selector')
    thumbnail_url = disc.css('.selector')
		
    encoded_link = URI.encode(link)
    doc = Nokogiri::HTML(open(URI.parse(encoded_link)))
		
    desc = doc.css('.selector')
		
    # weights gives us a jumbled number like - - - 151-175
    weights = doc.css('.selector')
    # this grabs the first 3 digits
    low = weights.scan()[0,3].join.to_i
    # this grabs the last 3 digits
    high = weights.last(3).to_i
		
    image_url = doc.css('.selector')
		
    # use these scraped values to create the Disc object on each iteration
    Disc.create(
      name: name,
      description: desc,
      disc_type: type,
      thumbnail_url: thumbnail_url,
      image_url: image_url,
      low_weight: low,
      high_weight: high
    )
  end
end
```

You can see that I have set up variables for the properties I want, then I create an object using those variables.

Under that method, I set up a simple logging system to keep track of how long the task takes to run. Then I loop through the `urls` hash and call the `scrape_discs` method for each `type`/`url`.

```
log = ActiveSupport::Logger.new('log/scraper.log')
start_time = Time.now

urls.each do |type, url|
  scrape_discs(type.to_s, url)
end

end_time = Time.now
duration = end_time - start_time
formatted_time = Time.at(duration).utc.strftime("%H:%M:%S")

log.info "Task completed."
log.info "Time elapsed: #{formatted_time}"
log.close
```

Now, when we run `rake scraper:scrape` it will create a new log file(or add to it if it already exists), set a start time, loop through our `urls` hash and gather all the information we need, then set an end time and calculate the duration of the task. Pretty cool!

## Part Three

Now how do we use all this data?! We'll need a few things:

1. Our rails backend needs a route (or routes) for Angular to hit
2. This route should serve up JSON data for the discs
3. An Angular Service/Factory to call our rails API and gather the data.

In `routes.rb` I'll set up an `api` scope, so that our routes will be something like `/api/route`. Then I'll just use `resources` to set up the common routes for discs.

```
scope "/api" do
  resources :discs
end
```

Now in `discs_controller.rb` I'll quickly set up an index action and tell it to render JSON.

```
def index
  @discs = Disc.all

  render json: @discs 
end
```

In order for this to work, I had to set up active_model_serializers with a discs serializer.

In terminal:

```
rails g serializer disc
```

This sets up a serializer in `app/serializers/disc_serializer.rb`. We'll need to go in this file and tell it which attributes you want to pass through.

```
class DiscSerializer < ActiveModel::Serializer
  attributes :id, :name, :description, :disc_type, 
             :thumbnail_url, :image_url,
             :low_weight, :high_weight, :selected_weight
end
```

Now if we navigate to `/api/discs` we see a beautiful list of JSON objects! Awesome. We'll use this in the next step.

### Creating an Angular Factory to access the Rails backend

To call the rails API, we'll be using Angular's `$http` service. `$http` is used to generate an HTTP request and returns a promise.

Let's setup the factory. We'll pass in `$http` as a dependency.

```
(function () {
  'use strict'
  
  angular
    .module('discCaddy')
    .factory('DiscFactory', ['$http', function($http) {

      return {
        getDiscs: getDiscs
      }

      // hits the #index method in discs_controller.rb
      function getDiscs() {
        return $http.get('/api/discs')
                         .then(handleResponse)
      }

      function handleResponse(response) {
        return response.data
      } 

    }])

}());
```

This factory provides a function called `getDiscs`, which will use `$http` to call our rails API, which will hit the `#index` method in `discs_controller.rb` and return a response object that contains all our JSON data. Pretty sweet! You can obviously set this up to get an individual object, to create objects, or much more.

There are a few ways you could use this Factory function, but in this case I chose to load it in a `resolve` property in the `discs` state. This means when the `discs` state gets called, it will call our Factory, and the resolved property will then be injected into the appropriate controller (in this case, `DiscsController`).

The route looks something like this:

```
.state('discs', {
  abstract: true,
  url: '/discs',
  templateUrl: 'discs/discs.html',
  controller: 'DiscsController as vm',
  resolve: {
    discs: function (DiscFactory) {
      return DiscFactory.getDiscs()
    }
  }
})
```

Now the controller will have access to a `discs` property that contains all of our disc data. 

It's easy to see how powerful this can be. And on that note, I'll wrap up this post.

Looking back, the whole project was a lot less daunting than it was at the beginning. This post is already long-winded, and I could probably at least double it with all I've learned in this project. Authentication with Angular was a trip, and could definitely warrant it's own blog post (or several!). 

It's easy to get Angular working on Rails, and not much harder to get Angular to hit the backend API. What you do with the data is up to you, but hopefully this post shed some light on how to get started.
