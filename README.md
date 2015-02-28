# best-practices-rails
Just a collection of some of the Rails best practices I have found over the past year since learning Rails

#Overview 
There is a lot abour Rails that I am still very new to. For the most part these guides wont go to far into the inner workings of rails, instead looking at the little things that you could do to better your code. 

#How it works
In this README alone, I go over every thing that is in this guide. If you want a more in depth explanation
simple go into the approraite file to learn more

#Delegations
delegate the fields between your models
	
	#avoid ----------------------

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

	#recommended -------------------

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
	

#Using scopes
Push as much of your database querys into their corresponding model as possible
	
	#avoid ----------------------

	#/controllers/votes_controller.rb
	class VotesController < ApplicationController
		def index
			@votes_for_current_ballot = Vote.where(ballot_id: 12, election_id: 14)
			@votes_not_counted = Vote.where(not_counted: true)
			@votes_counted = Vote.where(not_counted: false)
			@votes_before_now = Vote.where('created_at <= ?' , Time.now )
		end 
	end 

	#recommended -------------------

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

The scopes are not the only way to handle searhers. a scope of
```
scope :counted -> {where (not_counted: false)}
```
is the exact same as 
```
def self.counted
	where(not_counted: false)
end 
```

#Using Modules 
This is a great one. Its pretty obvious that you should make your Models as Fat as possible. The problems occurs when your models
suddenly get a little to fat. You will start noticing things that are catoriable with in the model. for example all the methods involving finders
can be moved to a module. same with generic and repetitive proccessing as well as validations through the help of concerns

	#avoid ----------------

	#/models/report.rb
	class Report < ActiveRecord::Base

		def export_to_pdf
			#implementation...
		end

		def export_to_csv
			#implementation...
		end

		def export_to_something_else
			#implementation...
		end 

		def self.find_by_date_range
			#implementation...
		end 

		def self.find_by_owner (owner)
			#implementation...
		end

		def self.advanced_search (fields, option ={})
			#implementation...
		end 

		def self.simple_search (fields)
			#implmentation...
		end

	end

As you can see, sooner or later this one model will become unmaintable. So the solution is to group the methods together and place them into modules, them import them back in.

	#recomended --------------

	#lib/report_searcher.rb
	module ReportSearcher

		def advanced_search (feilds, options = {})
			#implementation...
		end 

		def simple_search (feilds)
			#implementation...
		end

	end

	#lib/reports_finders.rb
	module ReportFinders
		def find_by_owner(owner)
			#implementation...
		end

		def find_by_date_range
			#implementation...
		end 

	end

	#lib/reports_exporters.rb
	module ReportExporters

		def export_to_something_else
			#implementation...
		end 

		def export_to_pdf
			#implementation...
		end 

		def export_to_csv
			#implementation...
		end

	end

	#/models/report.rb
	class Report < ActiveRecord::Base
		extend ReportFinders
		extend ReportSearcher
		include ReportExporter
	end 

The difference between include and extend is simple. they load in the methods from the modules as class and instance methods repspectively. in other words, if you method originally had self.method_name and was then placed inside of a module along with others, the that module would be loaded in to the Report Model through extend

#Use Select
If you only need 5 columns out of the 50 column table then white list them. It will greatly improve the performace of your app, as improper databse queries can really hurt

	#recomended-----------

	#lets say you only need the name, id and created_at columns from the user table
	class User < ActiveRecord::Base
		has_many :arcticles

		scope :current_users -> {where(fired: false).select(:name, :id, :created_at)}
	end

If you get the chance, compare select statments like theses to normal ones where you load in all data in side the rails console. Right next to the sql query you can see how long it took, I have seen the use of th select method reduce a query to 30% of the original time

#Use pluck
use :pluck instead of a combination of :select and :map. This is only efective for when you need the values from your querys in the form on an array. 
	#avoid -------------
	class Station < ActiveRecord::Base

		#this will return an array of all station_id's
		def self.get_station_ids
			select(:id).map{|station| Station.id}
		end

	end 

	#recomended---------
	class Station < ActiveRecord::Bse
		def self.get_station_ids
			pluck(:id)
		end
	end
not only is this less code but its also faster. See when you run the select command you are still loading in the active record objcts, this is a waste when right afterwards your simply extracking all the ID's. ;pluck allows you to retrieve those values with have to waste loading in the active records

#Use try
This is something that I never knew extisted untill recenttly and it goes a long way to cleaning up your code 
	#avoid ------------

	User.find(10).name

	#recomended -------

	User.find(10).try(:name)
the try method will return nil instead of raise an error and breaking the app when User.find(10) does not exist


#MetaProgramming
One of the interesting things about ruby in general is that Ruby is code is just text, nothing more and nothing less. What does this mean? It means you can have your code , create more code. The following example may not be releastice, but if you ever find your self in a similiar situation, definitly give meta programming a try.

	#avoid -------------

	#/models/vehicle.rb
	class Vehicle < ActiveRecord::Base
		
		def self.find_all_bmw
			where(manufacture: "bmw")
		end 

		def self.find_all_toyota
			where(manufacture: "toyota")
		end 

		def self.find_all_kia
			where(manufacture: "kia")
		end

		def self.find_all_honda
			where(manfacture: "honda")
		end

		..... etc.

	end 

What we have here is  a very repetitive set of finders and checkers. So lets try and and create these methods with some meta programming

	#recommended--------------

	#/models/vehicle.rb
	class Vehcile < ActiveRecord::Base

		MANS = %w(honda, kia, bmw, toyota)

		class << self

		MANS.each do |brand|
			define_method "find_all_#{brand}"
				where(manufacture: #{brand})
			end
		end

	end

As you can see, this is much much cleaner and just one of the cool things about Ruby. notice how the MANS array is a contsant (based on the first letter being a capital). also notice the class << self.
this is a way of making the following methods auto matically include self to the beggining of each method name

# Semantic View Helpers

When you want to apply some css and javascript to your views you can use the build in Rails semantic view helpers to greatly reduce the amount of ugly code that you have to write 
	#avoid--------------

	#/views/posts/index.html.erb
	<div class="posts" id="post_<%= @post.id %>" >


	#recomended----------
	
	#views/posts/index.html.erb
	<%= div_for @posts %>




	
