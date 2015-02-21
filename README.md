# best-practices-rails
Just a collection of some of the Rails best practices I have found over the past year since learning Rails

#Overview 
There is a lot abour Rails that I am still very new to. For the most part these guides wont go to far into the inner workings of rails, instead looking at the little things that you could do to better your code. 

#How it works
In this README alone, I go over every thing that is in this guide. If you want a more in depth explanation
simple go into the approraite file to learn more

#Delegations
delegate the fields between your models
	```
	#avoid

	#/models/author.rb
	class Author < ActiveRecord::Base
		has_many :articles
	end 

	class Article < ActiveRecord::Base
		belongs_to :author

		def get_author_name
			self.author.name
		end
	end 

	#/views/articles/index.html
	<%= @article.author.name %>

	#recommended

	#/models/author.rb
	class Author < ActiveRecord::Base
		has_many :articles

		delegate :name, :age, :gender, :position, to: :article
	end 

	class Article < ActiveRecord::Base
		belongs_to :author

		delegate :date, :topic, :title, to: :author
	end 

	#/views/articles/index.html
	<%= @article.author_name %> 
	```

#Using scopes
Push as much of your database querys into their corresponding model as possible
	```
	#avoid

	#/controllers/votes_controller.rb
	class VotesController < ApplicationController
		def index
			@votes_for_current_ballot = Vote.where(ballot_id: 12, election_id: 14)
			@votes_not_counted = Vote.where(not_counted: true)
			@votes_counted = Vote.where(not_counted: false)
			@votes_before_now = Vote.where('created_at <= ?' , Time.now )
		end 
	end 

	#recomended

	#/models/vote.rb
	class Vote < AcitveRecord::Base
		belongs_to: voter

		scope :for_current_ballot -> (id) {where (ballot_id: id)}
		scope :for_current_election - (id) {where (election_id: id)}
		scope :not_counted -> {where(not_counted: true)}
		scope :counted -> {where(not_counted: false)}
		scope :before_time -> (time) {where('created_at <= ?', time)}
	end 
	#/controller/votes_controller.rb
	class VoteController < ApplicationController
		def index
			@votes_for_current_ballot = Vote.for_current_ballot(12).for_current_election(14)
			@votes_not_counted = Vote.not_counted
			@votes_counted = Vote.counted
			@votes_before_now = Vote.before_time(Time.now)
		end 
	end 
	```

#Using Models 
This one is a great one. Its pretty obvious that you should make your Models as Fat as possible. The problems occurs when your models
suddenly get a little to fat. You will start noticing things that are catoriable with in the model. for example all the methods involving finders
can be moved to a module. same with generic and repetitive proccessing as well as validations throught the help of concerns




