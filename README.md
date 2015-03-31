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

# Find_each vs all

the find_each method does the exact same thing as .all.each, however it processess the model as a batch.
The size of the batch can be changed changed with the :batch_size options. Therefore if your iterating over a large number of records ( > 1000) then find_each is the  way to go. Otherwise, all works fine

	#recomended------------- 
	Billing.find_each do |bill|
		puts bill.amount
	end

#The N+1 Query 

A General room thumb is that you make one large query rather than 500 small ones. You can see exactly the kind and number of queries that you are making from with in the development log file. Lets first start by looking at the view

	<% @students.each do |student| %>
		<%= student.first_name %>
		<%= student.last_name %>
		<%= student.teacher.name %>
	<% end %>

Here we have a `:has_many` relation ship between `:teacher` and `:student`. This addvice does not affect view, If you have this in your view, your fine.

	class StudentsController < ApplicationControler
		def index
			@students = Student.all
		end 
	end 

So, to elobrate on the problem, when we make the initial query for the students that our (+1), and lets say that there are N records that `Student.all` returns. Thereforefore, when we look into the view we see the query that is being made to find the associated teachers name. This is the N query, (1 query for each of the N students). 

To fix this, we can use `includes`

	class StudentsController < ApplicationController 
		def index
			@student = Student.all.includes(:teacher)
		end 
	end

Looking at the SQL query, what this does is combines all the teachers and students where `teacher_id` on the student table matches up with `id` column of the teacher  table, it then takes the combined and loads them all in as a batch. Therefore, any all to @student.teacher happens instantly because they have already been loaded in!

#Has many through vs has and belongs to many 

It took me a while before I fully understood this myself. But the `has many through` is virtually the same as the `has and belongs to many`. The reason where you want to use one over the other is if the joing table has additional feilds outside of the two `:belongs_to` feilds. 

	class Owner < ActiveRecord::Base
		belongs_to :ticket
		belongs_to :user
	end 

	class Ticket < ActiveRecord::Base
		has_many :owners
		has_many :users, through: :owners
	end 

	class User < ActiveRecord::Base
		has_many :owners
		has_many :tickets, through: :owner
	end 

In this case the Owner model acts as the join model. There is a very likely chance the Owner model wont even have a corresponding controller, It acts as just a model for the join table that exists between Users and Ticket. As such, you can use them the same way.

To add a tickect to a user
	first_user  << random_ticket


# Using polymorphic associations
I truly belive this is one of those topics that needs much better documentation from the rails guide. It took me a lot of scouring and experminting before I fully understood this my self. 

So lets say you have two types of Events, one is a Online event and the other is a InPerson event. Now both of these events has many attendants, one way of implementing this is with two types of `attendant` models, first you have the `InPersonAttendant` and then you have the `OnlineAttendant`.Now this may seem sort of redundant to you and you might be asking yourself why not just build the `Attendant` model so that it has 2 foriegn keys (in_person_event_id, online_event_id)  and an `event_type`('InPerson','Online' ) field. And that is essentialy what a polymorphic association is !

Lets look at the code 
	class Person < ActiveRecord::Base
		belongs_to :attendant, polymorphic: true
	end 

	#notice how the Person class does not :belong_to either InPersonEvent or OnlineEvent

	class OnlineEvent < ActiveRecord::Base
		has_many :online_attendants, as: :attendant
	end 

	#notice how there is no direct refernce to the Person class

	class InPersonEvent < ActiveRecord::Base
		has_many :in_person_attendants, as: :attendant
	end 

	#notice how there is no direct refrence to the Person class

And there you have it ! A beautiful polymorphic association. Now notice how `attendant` servers as the middle man between the Person class and the InPersonEvent and OnlineEvent Classes. 

If your wondering about naming conventions its simple, the `:attendant` could have been anything we wanted so long as it matched in all three classes. The `:in_person_attendants` and `:online_attendants` could have also been anything we wanted. 

Now to use this association in action is simple. It works just like a `:has_many` association. 

	OnlineEvent.first.online_attendants

	InPersonEvent.first.in_person_attendants
	
#knowing SQL 

honestly, this is probobly one of the best peices of advice that you can get in this article. Most of these other things, are general suggestions practices, But this next one is a bit different. knowing SQL can greatly improve the performace of your site

Its very common to see people be lazy and use the rails active record methods when ever they need to interact with the database, but this has two problems. One it creates a lot of over head and two there is very large chace that things will happend in a very effecient way.

`Job.count`, This creates an SQl query to find the number job records there are in your database. 
`Job.length`, This will first load in all Jobs and then apply the length method. 

NOTE: If you ever wanted to know how the length function works, It goes a little like this. First we load in the data, and see how much memory it holds all together. It then looks at how much memory a single element takes up. It then divide!

Lets say you had a relation where  a Person `:has_many` Jobs. Now this means there are records in the Person class that do not have jobs. i.e Person.find_by_name(JoblessJoe).jobs will return an empty active::record:collection (an empty array more or less). Now lets say that you want a query to return only the Person records where person has at least one job. 

	scope :has_at_least_one_job, lambda {
		joins('LEFT OUTER JOIN job ON person.id = job.person_id')
		.group('person.id')
		.having('count(job.person_id) > 0')
	}

This a great example of how we can combine Rails and SQL to make super fast queries.

The above is just one of the ways that knowing SQL properly can help alot. This isnt a guide has to how to learn SQL, so unfortonatly we will be moving on

#Reduce your routes

A general rule in programming is to black list where ever we can. We do this for a lot of reasons, one it makes things faster as we saw with the `select` method and two, it goes a long way to unclutter your app. This idea can also be applied to Routes. 

lets say you were dealing with an app where the model `:device` does not have the ability to create, update or destroy. Instead they are ported in through some sort of  ruby script. Therefor there is really no need to include te routes to those ation.
	
	#avoid -------------

	resource :devices 

	#recomended ---------

	resources only:[:index, show] :devices


If you wanted to add some additional routes 
	
	resources :devices do 
		member do 
			get :battery
			post :update_with_pool
		end 

		collection do 
			get :locations
		end 
	end 













	
