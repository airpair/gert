# 1. Introduction
> If it's not tested, it's broken!

## 1.1 What are specs and why?
By using specs you do not only test a given piece of code for its pure functionality. No, you *specify* the behaviour of that piece of code or a whole system. Mostly you, as a developer, are given a brief description of a process or system (We all know that comprehensive user requirement specifications are a lie).
For example:
> I want users to sign up and confirm their account by e-mail.

So, what steps are involved and what behaviour is expected? Firstly, we need to *expect* that a user has an e-mail address after he has signed up and that he somehow confirms his account. **Hey, wait a second - we have our first spec here!**

```ruby
require 'rails_helper'
RSpec.describe User do
    it { is_expected.to have_db_column :email }
    it { is_expected.to validate_presence_of :email }
    
    context 'User Confirmation...' do
        it 'activates a User after Confirmation'
             expect{User.confirm}.to change{User.active}.from(false).to(true)
        end
    end
end
```
Basically that's what it is all about. Forge a fuzzy imagination into a specification and try to make it green.

## 1.2 Embrace BDD (Behaviour Driven Development)
When we first started to work with Ruby on Rails the terms *Behaviour Driven Development* and *Test Driven Development* have been used in nearly every tutorial and screencast. Soon we realized that testing your code is mandatory to build reliable software. Until then, testing was just optional and as many of you know very tedious. Even the setup of your test suite can be hard to accomplish if your application has a complex structure (consider dataflow logic, modeling logic or even time-based logic). Luckily Rails has a bunch of easy-to-use libraries (*gems*) to test your application.

A popular gem for testing your rails applications is *rspec*. Just add *the rspec-rails* gem to your Gemfile and run the provided installer and you are good to go to test your Rails application. As mentioned before, testing can be very tedious due to the fact that a lot of test-scenarios are repeated with a slightly different parameter or environment and even with rspec and the DRY philosophy repetition of code cannot be eliminated completely.

To bring up an example let’s consider the following ActiveRecord model:

```ruby
# == Schema Information
#
# Table name: foos
#
#  id               :integer          not null, primary key
#  title            :string(255)
#  bar_id           :integer
#  account_id       :integer
#  created_at       :datetime
#  updated_at       :datetime
#
class Foo < ActiveRecord::Base
    # === Relations ===
    belongs_to :bar
    # === Validations ===
    validates :title, presence: true, length: {maximum: 255}, allow_blank: false
    validates_presence_of :bar
end
```

This model has only one relation to another model called **:bar** and two validation methods that ActiveRecord provides.
The question is do we need to test this model and if we do what would the test or spec look like?

Well, we could rely on ActiveRecord and assume that it executes properly and that ActiveRecord has its own tests to make sure that those validation methods and relations are not broken and in fact ActiveRecord has those tests. So what we don’t want is to test ActiveRecord again. We want to abstract from that and capture the behaviour of our model into a test or spec. This captured behaviour can then be used to inform us about changes to a model and how those changes can possibly afflict other models.

## 1.3 The tedious task of writing simple tests/specs
Face it, writing simple tests that capture your Rails application behaviour can be boring and straight forward but it is mandatory for a reliable and stable piece of software. Let's bring up the Foo example from 1.2. For such a model we would write a simple spec like:
```ruby
describe Foo do
    it { is_expected.to belong_to :bar }
end
```
Such a spec would only cover that the belongs_to relation :bar exist. If we assume that your model has (and mostly it has) more than one relation we can take our basic structure and only change the relation name :bar into another relation. So basically we are just copy and paste our example with a slighlty modified environment. Now imagine you need to do this to a large rails application with more or less than 100 models you would go nuts in an instant.

# 2. Dude... just generate it!
## 2.1 Using a rails generator
If you are working with rails the following command is not unfamiliar.
```bash
rails generate ...
rails g ... #shortcut for generate
```
So how to write such a generator? It is fairly simple. Just create a file in the **lib/generators** directory of your Rails app and name this file according to the Rails conventions with **NAME_generator.rb**. Use the following basic structure for your generator.
```ruby 
class NAMEGenerator < Rails::Generators::Base
    source_root(File.expand_path(File.dirname(__File__)))
    
    def methodX
        ...
    end
end
```

After that you can invoke this generator by using the following command inside your rails app.


```bash 
rails g NAME
```
Notice that we pass only NAME and not NAMEGenerator as an argument to our command. As smart as Rails is it will infer the correct generator based on its name. The **rails g** command will then load the given generator and execute every method or line you have defined inside your class.
## 2.2 Generate model specs
#### Extract validations
Why should I use validations? Well... why not. Validations ensure that data which is persisted into your database is valid and therefore validations lead to an application which is reliable and stable even on incorrect user input.
Thanks to ActiveRecord we can easily access validators of a model by calling:
```ruby
Model.validators
# => [#<ActiveRecord::Validations::...>, ...]

```
Those validators can be used to generate specs like
```ruby
it { is_expected.to validate_presence_of :attribute }
```
So let's add a little more awesomeness to our generated validator examples and
 provide negative examples. What are negative examples? Simple! Examples which are violating the validation. Let's bring up an example again and consider the following validation.
 
```ruby
validates_length_of :attribute, minimum: 0, maximum: 100
```
Our **:attribute** can have an upper and lower bound to its length. A positive spec example would be

```
it { is_expected.to allow_value(Faker::Lorem.characters(100)).for :attribute }
```
And a negative example would violate the upper bound by +1. **(Same goes for the lower bound by -1)**
```
it { is_expected.not_to allow_value(Faker::Lorem.characters(101)).for :attribute }

```

#### Extract relations
ActiveRecord enables you to reflect on associations of a model. Using this reflection we are able to extract the following relation types:
 - belongs_to
 - has_one
 - has_many
 
Based on those extracted relations we are able to create simple spec examples like:
```ruby
it { is_expected.to belong_to :association_name}
it { is_expected.to have_one :association_name}
it { is_expected.to have_many :association_name}

```
Of course we utilize [shoulda-matchers](https://github.com/thoughtbot/shoulda-matchers) to generate beautiful and easy to read specs.
 
#### Extract nested attributes
Yes! we can even generate specs for nested attributes like.
```ruby
it { is_expected.to accept_nested_attributes_for :attribute }
```
by simply calling 
```ruby 
Model.nested_attributes_options.keys
```
and throwing the result into a spec example.
#### Extract database information (columns, indexes)
Need information about the database representation of your model? Easy!
To get the columns of your model call.
```ruby
Model.columns.map(&:name)
```
To get the indexes you need to... wait tl;dr:
```ruby 
ActiveRecord::Base.connection.indexes(Model.tableize.gsub("/", "_"))
```
#### Put them together and throw them into a template
Sounds harder than it is. To achieve this goal we utilize the ERB templating mechanism by creating a *model_spec_template.rb* where we can call our extraction methods on. Our template would look like this:

```ruby
require 'rails_helper'

RSpec.describe <%= @model.model %> do
  ...
  # === Relations ===
  <%= @model.belong_to_relations %>
  <%= @model.has_one_relations %>
  <%= @model.has_many_relations %>
  ...
end
```
Result: much model specs, such behaviour!

## 2.3 Generate controller or route specs
How can we receive automated information about a controller? Should we trigger our application and record what happens and capture those results into a spec? Well it could result in a very complex logic which is prone to errors. An easier approach would be to utilize the **routes.rb** from your application and infer information about your controller via this file. The general idea to receive information about a controller via the **routes.rb** would be to extract the controller name and actions from a resource then use this controller name and actions to find a route which matches together with a request method. 

If we find a match of the triple **(method, controller, action)** we can generate a spec which looks like this:
```ruby
	it { 
	    should route(:get,'/models/1').to(
	        {:controller=>"models", :action=>"show", :id=>1}) 
	}
	
	it { 
	    should route(:patch,'/models/1').to(
	        {:controller=>"models", :action=>"update", :id=>1}) 
	}
	
	it { 
	    should route(:post,'/models').to(
	        {:controller=>"models", :action=>"create"}) 
	}
	
	it { 
	    should route(:destroy,'/models/1').to(
	        {:controller=>"models", :action=>"destroy", :id=>1}) 
	}
	
	it { 
	    should route(:get,'/models').to(
	        {:controller=>"models", :action=>"index"}) 
	}
```
### Filters and actions
Another information we can process automatically are filters in controller. To receive all filters from a controller just call
```ruby
ModelController._process_action_callbacks
```
and use them to generate specs like
```ruby 
it { should use_before_filter(:some_filter) }
it { should use_after_filter(:some_filter) }
it { should use_around_filter(:some_filter) }
```

# 3. Capture your Rails app
So much for the technical details, now I know how to write a Generator for Specs, wonderful... really! But how does this help me in my everyday live?
## 3.1 Regression testing
It's monday morning, the client calls "I don't want the features provided by **bar** model anymore, delete it". Okay it's early, the client was quite annoying and you decide to simply delete that model and be done with it. Later that week, again a call, again the client "Your app is crashing when I try to use the features of the **foo** model!". Did you remember? In section 1.2? The **foo** model belongs to **bar**. And as always the worst case happened, your app is indeed crashing. Probably just about a minute ago you read the definition of the **foo** model and already forgot that it *belonged to* **bar**. Imagine your glimpse at that model was a week ago, a month, a year. 

That's why we came up with the idea to automate some very simple tests to keep track of our ever changing application. Regression testing is a type of software testing that seeks to uncover new software bugs, or regressions in existing functional and non-functional areas of a system after changes, such as enhancements, patches or configuration changes, have been made to them.
## 3.2 Introducing GeRT
Who is this GeRT and why is he so awesome? GeRT means **Generated Regression Testing**. GeRT is everything we explained above distilled into one punchy word. GeRT.

With the help of our generators for model tests, controller tests, routing tests and so on you can skip the initial hassle when making changes to existing components of your system or when implementing new ones. 
A typical workflow with GeRT would look like the following:

 1. Make any kind of changes to your application, like removing a validation, adding an association, changing a route, adding another action to your controller.
 2. When you're done with your first set of changes, run the **rail g regressor:model** or **rails g regressor:controller** command and track all changes and side effects you just created in the specs you have just generated. 
 3. Run the specs and check if you made any mistake. If yes, fix it! If no, *goto* 1. 

## 3.3 GeRT with benefits
With GeRT you won't become a fan of TDD or BDD. With GeRT you won't like writing tests. With GeRT you won't find esoteric bugs in your business logic. 
**But GeRT will give you one thing: You will make less trivial mistakes. And those trivial bugs are the hardest to find.**

# 4. TL;DR Just give me my gem!
Alright here you go! I've packed all this into a gem and published it to github: [regressor](https://github.com/ndea/regressor/ "Get it now!"). 
Contributions, stars, forks, watches and issues... well issues maybe not so much are appreciated.

# 5. Conclusion
Since we are lazy developers who, as most of you, don't like to write tests this idea of generating tests or specs is an awesome way to develop more reliable software and get immediate feedback through our generated test suite of changes applied to the behaviour of the application. 