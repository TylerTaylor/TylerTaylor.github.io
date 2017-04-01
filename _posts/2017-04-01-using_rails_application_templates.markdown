---
layout: post
title:  "Using Rails Application Templates"
date:   2017-04-01 17:52:19 +0000
---


A lot of my Rails apps start out the same way. Generate the app - generate or create a welcome controller - create a view - set up routes - set up controller actions - set up Devise or another authentication gem - bundle - git init - you get the point.

Rails provides us something awesome called _Application Templates_. Application templates are simple Ruby files containing DSL for adding gems, initializers, and running commands for a new (or existing) Rails project.

Does this mean we can set up an Application Template to do everything in the list above? Absolutely.

Simply create a `.rb` file for the template. I'll call mine `rails_template.rb` and put it in my workspace directory, where I generally create my new apps.

These are the settings I tend to use. You can change any part of this to better suit your needs.

```
gem 'devise'
gem 'pg'

generate(:controller, "welcome index")
route "root to: 'welcome#index'"

run 'rails generate devise:install'
run 'rails generate devise User'

rails_command("db:migrate")

# Remove sqlite3 and turbolinks gems
gsub_file "Gemfile", /^gem\s+["']sqlite3["'].*$/,''
gsub_file "Gemfile", /^gem\s+["']turbolinks["'].*$/,''

gem_group :development, :test do
  gem 'guard', require: false
  gem 'guard-livereload', require: false
end

after_bundle do
  git :init
  git add: '.'
  git commit: "-a -m 'Initial commit'"
end
```

When you want to use the template, you just generate a Rails app with a `-m` flag and the name of your template:

`$ rails new app_name -m rails_template.rb`

Barring any typos or errors, this should run and create the Rails app with all the specified adjustments. In this case, the template will:

* Add the specified gems - Devise and Postgres
* Remove specified gems - SQLite3 and Turbolinks
* Set up a Welcome controller with an index action, and a matching index view
* Set root route to 'welcome#index'
* Run the Devise installer
* Generate a User model for Devise
* Migrate the database
* Add guard and guard-livereload gems to `:development, :test` group
* After bundling - initialize git repo, add files, and commit

All of this from typing one command in the terminal!

There's a small caveat with this setup: The OCD in me has to go into the Gemfile to rearrange the gems (which get added to the bottom), and currently this adds another `:development, :test` group instead of appending to the one Rails gives us. That said, the convenience and time-savings still make this worth using for me.

Overall I think this is a really cool way to get a project going in a very quick manner. If your apps generally have the same setup, I'd highly recommend looking into [Rails Application Templates.](http://guides.rubyonrails.org/rails_application_templates.html) I think it's worth it even if you only use the git commands. Happy coding!
