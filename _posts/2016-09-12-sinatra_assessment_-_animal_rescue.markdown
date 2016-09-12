---
layout: post
title:  "Sinatra Assessment - Animal Rescue"
date:   2016-09-12 15:15:01 +0000
---


Setting out to build my first Sinatra project from scratch was actually less daunting than I thought it would be. The hardest part for me was coming up with an idea. We couldn't do a blog or twitter clone, but we needed to create some sort of CRUD app.

A buddy of mine just happened to be complaining about the process of finding a dog to adopt. LIGHT BULB! I'll make a web app where people can view multiple animal shelters and the animals they have available. 

I started out in a frenzy, trying to get the animal and shelter classes built. I had completely forgotten that there needs to be users as well. So I simply added a User class, then made it a requirement to be logged in to do certain things. I built a helper method ```logged_in?``` that checks that for us.

# The Models
- **Animal**: *belongs_to* shelter. This was a little confusing at first. I had them belonging to the user, but that doesn't quite make sense because they wouldn't be in a shelter if they had an owner.

- **Shelter**: *has_many* animals, *belongs_to* user. A user is responsible for creating a shelter object. They can't be created without a user being signed in.

- **User**: *has_many* shelters, *has_many* animals *through* shelters. 

# Users
An instance of the User class ```has_secure_password```, which requires the class to have a password_digest attribute. This is from ActiveRecord, which will do its magic and encrypt (with bcrypt) the users password. If the user leaves any fields empty when signing up, they will be redirected to the sign up page, with a flash message instructing them to fill out every field.

If the fields are all filled out, when the user submits the form our ```post '/signup'``` route will create the new user, and set the session[:user_id] equal to the user's ID. Boom, we have a logged in user! To log them out, our ```get '/logout'``` route simply calls ```session.destroy``` to clear out the session info. Then we give a nice flash message to confirm the user has logged out.

I'm sure the user validation could be taken to a whole new level, but for now I'm happy with it.

I may add user show pages later, but it didn't seem crucial to the project so I've skipped that for now.

# Styling

I'm 'happy' to say that most of the issues I had on this project were just styling issues. Couldn't get the footer to stick to the bottom to save my life, now that just seems absurdly simple. ```position: fixed, bottom: 0px```

I used a simple bootstrap layout. I gave the shelter and animal 'cards' a class of ```col-md-4``` to have a nice grid layout. 

The layout.erb file does some neat conditional checks to decide what to display. If there is no one logged in (checked via ```#logged_in?```), there will be a login form in the nav bar. If someone is logged in, there is a personalized greeting in the navbar, and a sign out link.

Each index/show page also does conditional checks to see whether the shelter or animal belongs to the current user. If it does, links to edit that item will appear.

The layout file is also where I put the flash message container. I set it up with an if/else statement, to check whether the flash message is an error or success. Paired with some CSS tricks, I have it set up to be red if it's an error, and green if it's a success. 

```
<% if flash.has?(:error) %>
  <div class="error-message">
    <p><%= flash[:error] %></p>
  </div>
<% elsif flash.has?(:success) %>
  <div class="success-message">
    <p><%= flash[:success] %></p>
  </div>
<% end %>
```

That was such a small detail but felt really cool to implement.

Overall I had a blast building this project, and have grown to really love Sinatra in the process. It is a very lightweight framework to build off of, and I'm sure I'll be using it for other projects down the road. You can view all of the code for this project on github: https://github.com/TylerTaylor/sinatra-final-animal-rescue
