---
layout: post
title:  "Building my first CLI gem, Sport-Stats"
date:   2016-08-17 11:02:08 -0400
---

Starting a new project from a blank slate was pretty daunting. Finishing, however, was quite the feeling. Cue the Rocky song!

I knew I wanted to make something I would actually use. I'm a big sports fan, so naturally I landed on a sports app. Sport-Stats provides quick and easy access to NBA, NFL, and MLB scores, as well as rosters and player information for each team.

At first, I was making life way too difficult for myself. I wasn't properly using objects! I had an elaborate scraping maze going on, and it got to the point where I lost track of what was controlling what. All my information was being scraped and stored in arrays and hashes. To display the information, I was going through several loops, and formatting was a nightmare.

After spending a few days spinning my wheels, I realized I had it set up all wrong! I needed a Team class to hold the teams, and a Player class to manage the players. That way I could cleanly format everything, and use commands like team.name, player.age, etc.

When setting up the Player class, I struggled a bit with all the different parameters. They all have similar parameters, but each league has a few specific to that sport. I remembered a lesson where we had to pass in an array of arguments, and had a total AHA moment.

This is when I learned about the magical ```||=``` operator in ruby. 

```
class SportStats::Player

  attr_accessor :no, :name, :pos, :age, :height, :weight, :college, :salary,
                :bat, :thw, :birth_place, :exp

  @@all = []

  def initialize(params)
    @no ||= params["NO."] || params["NO"]
    @name ||= params["NAME"]
    @pos ||= params["POS"]
    @bat ||= params["BAT"]
    @thw ||= params["THW"]
    @age ||= params["AGE"]
    @height ||= params["HT"]
    @weight ||= params["WT"]
    @exp ||= params["EXP"]
    @college ||= params["COLLEGE"]
    @birth_place ||= params["BIRTH PLACE"]
    @salary ||= params["2016-2017 SALARY"] || params["SALARY"]
    @@all << self
  end

  def self.all
    @@all
  end

end
```

All 3 of the sports have number, name, position, height/weight,and college columns. That leaves us with 6 other attributes we may or may not have. That's where ```||=``` comes in! If the value to the left is nil or false, it will check the value on the right. If the value on the right exists, it sets the variable to that value. If not, it remains nil.

The params that get passed into the Player class look like this:
```
{"NO."=>"21", "NAME"=>"Ian Clark", "POS"=>"SG", "AGE"=>"25", "HT"=>"6-3", "WT"=>"175", "COLLEGE"=>"Belmont", "2016-2017 SALARY"=>"$980,431"}
```

So if the hash contains keys that match our instance variable names, the value will get assigned to the proper variable. On that note, another handy thing I learned in this project is the .zip method. The .zip method will merge two arrays into either an array or a hash. I chose hash. 

Here's my roster scraping method.

```
def self.scrape_roster(input, doc)
	doc.search(".standings-row td").each do |x|
		if input == x.css("span.team-names").text # if x is equal to the team name input
			team_page_link = "http://espn.com#{x.search('a').attr('href').value}"
			team_page = Nokogiri::HTML(open(team_page_link)) 
			@players = []

			team_page.css("span.link-text").each do |text|
				if text.text == "Roster"
					link = Nokogiri::HTML(open(text.parent.attr('href')))

					@categories = [] # These will be used for player attributes
					link.search('tr.colhead').first.children.each do |cat|
						@categories << cat.text
					end

					link.css('tr.evenrow, tr.oddrow').each do |word|
						player_info = [] # name, age, etc, player attr values
						word.children.each do |word|
							player_info << word.text
						end

						player_line = Hash[@categories.zip(player_info)]
						@players << player_line
					end
				end # end if
			end # end .each
		end # end if
	end # end doc.search
end
```

You can see I set up a categories array, and a player info array. Nokogiri scrapes the specified elements and puts them in their proper container. This is basically how I had the whole program set up before. It would collect the elements like above, then I'd use a loop to print each value out. I had to do this for each league, and each roster. It was very, very messy. Now it neatly combines the player information, then passes it to a make_players method that creates each individual Player object with proper attributes.

```
def self.make_players(team, doc)
	self.scrape_roster(team, doc) # scrape_roster returns @players with a hash of their info
	@players.each do |player|
		SportStats::Player.new(player)
	end
end
```

So my biggest challenge at first was getting the information to display properly. This was basically impossible with all the looping, and I realized I needed proper objects. This also lead me to find a very handy ruby gem called Command Line Reporter (Actually, another student named Brian Reynolds pointed me in that direction. Shout out to Brian!). CLR does a lot of neat things, but most importantly for me, it provides us with a nice looking table for our information. 

![](http://i.imgur.com/KMwoIMZ.png)

Here's how I set that up (note: you have to extend CommandLineReporter)

```
def self.display_stats
	table(:border => true) do
		row do
			column('', :width => 2, :align => 'left')
			column('Team', :width => 25)
			column('W', :width => 5)
			column('L', :width => 5)
			column('PCT', :width => 5)
			column('GB', :width => 5)
			column('HOME', :width => 5)
			column('ROAD', :width => 5)
			column('DIV', :width => 5)
			column('CONF', :width => 5)
			column('PPG', :width => 5)
			column('OPP PPG', :width => 5)
			column('DIFF', :width => 5)
			column('STRK', :width => 5)
			column('LAST 10', :width => 5)
		end
		SportStats::Team.all.each.with_index(1) do |team, index|
			row do
				column("#{index}")
				column(team.name)
				column(team.wins)
				column(team.losses)
				column(team.pct)
				column(team.gb)
				column(team.home)
				column(team.road)
				column(team.div)
				column(team.conf)
				column(team.ppg)
				column(team.opp_ppg)
				column(team.diff)
				column(team.strk)
				column(team.l10)
			end
		end
	end
end
```

We set up a table, and give it a border. The 'row do' line creates a row, and the columns, well, create columns. 
Here, I have hard coded the values of the categories. This works for the general overview of the teams, but not for the rosters, since they have different categories.

```
def self.print_roster
    obj = SportStats::Player.all.first # grab a player object to get the titles
    titles = []
    obj.instance_variables.each do |var| # grab all instance variables this object has
      if obj.instance_variable_get(var) != nil
        var = var.to_s 
        var[0] = '' # take off the @
        titles << var.upcase
      end
    end

    table(:border => true) do
      row do
        titles.each do |title|
          if title == "NAME" || title == "COLLEGE" || title == "BIRTH_PLACE" ||
            title == "SALARY"
            column(title, :width => 20)
          else
            column(title, :width => 6)
          end
        end
      end # end row
      SportStats::Player.all.each do |player|
        row do
          player.instance_variables.each do |var|
            if player.instance_variable_get(var) != nil
              column(player.instance_variable_get(var))
            end
          end
        end # end row
      end
    end # end table
  end
```

So in order to deal with the varying categories, I set up an empty array for 'titles'. I grab a Player object (we're already in the right league when it gets to this point of the code) and need to get the instance variables from that object. If the instance variable is nil, we don't add it to the titles array.

Then we start our table. In the first row, we loop through each title and create columns. If the title is a certain word, we make the column wider to accomodate for longer names/words. In the end, we get this:

![](http://i.imgur.com/bbKRhaY.png)

Here's a video showing the CLI in use (might need to go full-screen to see the text):
<iframe width="720" height="500" src="https://www.youtube.com/embed/JD98hxZAfEI" frameborder="0" allowfullscreen></iframe>

There was a bit of head scratching on this project, mostly due to scraping a bunch of unlabeled ```<td>``` elements. Overall, it was a very fun project, and has me excited for what's to come! 

You can check out the gem on github at https://github.com/TylerTaylor/sport-stats and feel free to message me if you have any questions.
