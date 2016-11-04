---
layout: post
title:  "Rails g new BlogPost"
date:   2016-11-04 15:02:40 -0400
---


FilmSpot is a simple web app to keep track of movies you've watched, and whether you liked them. I watch a ton of movies, to the point that it's hard to remember everything I've seen, much less my thoughts on all of them. Thus, FilmSpot was born.

# First Hurdle
 
I knew I would have models for users, movies, viewings (an instance of a user watching a movie), and ratings. After digging in, I realized that ratings could just be an attribute of viewings, so I refactored. I also included models for actors, and directors, but those aren't as integral to the app as of right now.

```
class User
  has_many :viewings
end

class Movie
  belongs_to :director
  has_many :viewings
  alias_attribute :viewers, :users
  has_many :users, :through => :viewings
end

class Viewing
  belongs_to :user
  belongs_to :movie
end

class Actor
  has_many :roles
  has_many :movies, :through => :roles
end

class Director
  has_many :movies
end
```

Working on the Movie class, I learned about alias_attribute. I was playing in the console and calling Movie.users just seemed weird. Movie.viewers sounds much better. 

The viewings table is a join table for Users and Movies. It has an attribute for rating, which the user will submit (one of the project requirements).

# Second Hurdle

Let's talk about nested resources, and nested forms.

I have it set up where you can add a new movie directly from a director's show page. I knew I wanted the path to be something like "/directors/1/movies/new" so how do we get that? Nested resources! When one resource is logically the child of another resource (ie, this movie *belongs to* this director), we can use nested resources.

```
resources :directors do
  resources :movies
end  
```

This creates several helper routes for us, namely new_director_movie_path.

Now the new movie form needs to take in director attributes. Let's nest a form! We'll use fields_for on our form builder (f) and create fields for our director. If we already have a director in mind, which in this case we do, we need to tell the form about it. So the new action would look something like this:

```
def new
  @movie = Movie.new
  @director = @movie.build_director
  # actors built here later
	
  if params[:director_id]
    @director = Director.find(params[:director_id])
  end
end
```

And the form. Notice how we check ```<% if params[:director_id] %>``` and pass in the director's name as a value if one is present. @director is a blank director object in the new action, unless the params come in with a director_id, then @director should have a name. 

```
<%= form_for(@movie) do |f| %>

  <div class="field <%= 'field_with_errors' if @movie.errors[:title].any? %>">
    <%= f.label :title %>
    <%= f.text_field :title %>
  </div><br>

  <%= f.fields_for :director do |d| %>
    <%= d.label :director %>
  
    <% if params[:director_id] %>
      <%= d.text_field :name, value: @director.name %>
    <% else %>  
      <%= d.text_field :name %>
    <% end %>
  <% end %>

  <br><br>
  <div class="actions">
    <%= f.submit %>
  </div>
<% end %>
```

Okay, all is good! But our movie model doesn't know what to do with this new director information. We need to define a method in our movie model to accept the nested attributes.

```
def director_attributes=(director_attribute)
  self.director = Director.where(:name => director_attribute[:name]).first_or_create
end
```

This method accepts the desired attributes as params, and searches for a director with a matching name. ```first_or_create``` will either find a matching director object or create a new object. Then we set the result to self.director. Now when a new movie gets submitted, a director is either created or updated.

# Third Hurdle

We were required to include a class level ActiveRecord scope method and create a URL to see the working feature. Since this is a list of movies, I went with 'most viewed.'

routes.rb:
```
get 'movies/most_viewed' => 'movies#most_viewed_movies', as: 'most_viewed_movies'
```

Now we have our URL, and a URL helper, most_viewed_movies_path.

This was a major struggle for me. I tried several different ActiveRecord queries but couldn't quite figure it out. Finally, I came across an option for belongs_to associations, called counter_cache. Counter_cache keeps track of objects *belonging to* another object. So in this case:

```
class Viewing
  belongs_to :user
  belongs_to :movie, counter_cache: true
end
```

The viewing instance *belongs to* a movie. Now, each time a viewing occurs it will add to the viewings count of the movie. In order to keep track of that, I added a "viewings_count" column in the movies table. In the viewings#create method, I built an instance of a user's viewing, passing in the movie_id, user_id, and rating. Then .reset_counters runs an SQL count query and sets the count to the right number. Then ```self.most_viewed``` returns a collection of viewings, sorted by their view count.

```
def self.most_viewed
  order('viewings_count DESC')
end
```

I had a lot of fun building this project, despite all the times I wanted to throw my laptop off of the roof. I feel like I learned a ton just from this one project, and am eager to tackle more! I plan to add more features to this as my knowledge expands, including friends, pictures, an improved rating system, and movie suggestions.

Happy coding!
