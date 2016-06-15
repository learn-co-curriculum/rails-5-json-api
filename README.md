# Building a Rails 5 API + JSON API

In an [earlier post](http://www.thegreatcodeadventure.com/building-a-super-simple-rails-api/), we built a super quick and super simple Rails API using the Active Model Serializer gem and the Active Model Adapter. 

There's been a lot of growth lately, however, around JSON API. 

### What is JSON API?

So, what is JSON API and what's so great about it?

From the docs...
> JSON API is a specification for how a client should request that resources be fetched or modified, and how a server should respond to those requests.

>JSON API is designed to minimize both the number of requests and the amount of data transmitted between clients and servers. This efficiency is achieved without compromising readability, flexibility, or discoverability.[*](http://jsonapi.org/format/#introduction)

To learn a bit more about what's so great about it, check out this article from [Programmable Web](http://www.programmableweb.com/news/new-json-api-specification-aims-to-speed-api-development/2015/06/10)

#### How does it format data?

JSON API serialized data should contain the following:

* The root level of the JSON API response is a JSON object.
* This object will contain a top-level key of `data`
* The `data` key points to either a single JSON object representing a record, or a collection of JSON objects representing records. 
* The JSON object representing a single record will contain the following:
  * The `id` of the record
  * The `type` of the record, i.e. the name of the resource, like "posts", or "cats". 
  * A key of `attributes`, which points to a JSON object containing key/value pairs representing that record's attributes. The attributes that appear here, if any, are determined by the way in which you serialize your data. 
  * An optional key of `relationships`, which will point to a JSON object that describes the relationship between the resource and other JSON API resources.
* In addition to the top-level key of `data`, you may see a top-level key of `included`, which will point to a collection of JSON objects representing records associated to the requested records. 

To learn more about the JSON API format, check out [this section of the docs](http://jsonapi.org/format/).

### Implementing JSON API in a Rails 5 API

Now that we're all convinced that JSON API is the way to go, and we have a basic understanding of the manner in which it formats data, let's build it!

#### The App

We'll be building a simple Rails 5 API that serves data regarding cats and their hobbies (yes cats have hobbies). This API is being developed in response to the clamor from Flatiron students for *more* cat-themed projects. [Some](http://hostiledeveloper.com/) [people](http://beatscodeandlife.ghost.io/) don't believe that there is such a clamor, but there is. Trust me. 

<iframe src="//giphy.com/embed/FyGHRHcD3RzCo" width="480" height="480" frameBorder="0" class="giphy-embed" allowFullScreen></iframe><p><a href="http://giphy.com/gifs/catvidfest-cat-grumpy-alexander-lansang-FyGHRHcD3RzCo">via GIPHY</a></p> 

So, our app has two main resources: cats and hobbies. A cat has many hobbies and a hobby has may cats. We have a cat-hobbies join table to accommodate that many-to-many relationship. 

You can view the completed code [here](https://github.com/learn-co-curriculum/rails-5-json-api-example-app), by the way. 

Let's begin!

###### Step 1: Getting Started

First, `gem install rails --pre` to get the latest version of Rails 5. Then run `rails new catbook --api --database=postgresql` to generate your very own Rails 5 API. If you're working with Rails 4, that's okay too. Just run `gem install rails-api` and run the same command to generate your app. 

Great, go ahead and `cd` into the directory. Add the following gems to your Gemfile:

```ruby
gem 'active_model_serializers'
gem 'rack-cors'
```

And, if you're working with Rails 4

```ruby
gem 'rails-api'
```

Go ahead and bundle install.

Now, we need to set our API's adapter to JSON API. Create a file, `config/initializers/active_model_serializer.rb` and set the adapter here:

```ruby
ActiveModelSerializers.config.adapter = :json_api
```

This will tell Rails to serialize our data in the JSON API format. 

We also have to tell our app to accept the JSON API mime type when receiving data (for when our client `POST`s data to the server). In the same file, add the following:

```ruby
api_mime_types = %W(
  application/vnd.api+json
  text/x-json
  application/json
)
Mime::Type.register 'application/vnd.api+json', :json, api_mime_types
```

Lastly, don't forget to set up CORS. I use the `rack-cors` gem. If you're working with Rails 4, simply include the following in your `config/application.rb`:

```ruby
...
config.middleware.insert_before 0, "Rack::Cors" do
  allow do
    origins '*'
    resource '*', :headers => :any, :methods => :get, 
      :post, :delete, :put, :patch, :options, :head
    end
  end
...
```

In Rails 5, simply comment in the code in `config/initializers/cors.rb` and set your `methods:` to the collection above.

###### Step 2: Domain Model

Generate a resource for Cat, Hobby and Cat Hobbies. Define your migrations to give Cat the following attributes:

```ruby
class CreateCats < ActiveRecord::Migration[5.0]
  def change
    create_table :cats do |t|
      t.string :name
      t.string :breed
      t.string :weight
      t.string :temperament
      t.timestamps
    end
  end
end
```

Hobby:

```ruby
class CreateHobbies < ActiveRecord::Migration[5.0]
  def change
    create_table :hobbies do |t|
      t.string :name
      t.timestamps
    end
  end
end
```

and, Cat Hobbies:

```ruby
class CreateCatHobbies < ActiveRecord::Migration[5.0]
  def change
    create_table :cat_hobbies do |t|
      t.references :cat, index: true
      t.references :hobby, index: true
      t.timestamps
    end
  end
end
```

Then, set up your models with the following associations:

```ruby
# app/models/cat.rb

class Cat < ApplicationRecord
  has_many :cat_hobbies
  has_many :hobbies, through: :cat_hobbies
end
```

```ruby
# app/models/hobby.rb

class Hobby < ApplicationRecord
  has_many :cat_hobbies
  has_many :cats, through: :cat_hobbies
end
```

```ruby
# app/models/cat_hobby.rb

class CatHobby < ApplicationRecord
  belongs_to :cat
  belongs_to :hobby
end
```

###### Step 3: Routes and Controllers

We'll namespace our routes in the following way:

```ruby
Rails.application.routes.draw do
  namespace :api do
    namespace :v1 do
      resources :cats, except: [:new, :edit]
      resources :hobbies, except: [:new, :edit]
    end
  end
end
```

And our controller file structure should follow that pattern:

```bash
├── app
    ├── controllers
        ├── api
            └── v1
                ├── cats_controller.rb
                └── hobbies_controller.rb
        ├── application_controller.rb
```

Notice that there are no routes defined for Cat Hobbies. That is because the users of the client-side app that will consume this data will need to show cats and their associated hobbies, as well as hobbies and their associated cats, *but not Cat Hobbies*. As far as our user is concerned, there's no such thing. That model and table only exist to allow us to implement the many-to-many relationship between cats and hobbies. 

###### Step 4: Serializers and JSON Rendering

When the client requests cat records, we want to include the related hobby records as well. In fact, we want to *side load* that data. This means that we will include the entire record of a given associated hobby or hobbies, when the client requests cat records. 

**Why Side Load?**

Well, our client needs to display both cat and associated hobby data to the user. Since this data needs to be displayed together, it makes sense for our API to serve it together. If we side load that data, i.e. include hobby records when the client asks for cats, the client won't have to make additional API requests when it needs a cat's hobbies. 

**Cat Serializer**

So, let's define our Cat Serializer to serialize a cat's attributes, along with its associated hobbies:

```ruby
class CatSerializer < ActiveModel::Serializer
  attributes :id, :name, :breed, :weight, :temperament
  has_many :hobbies
end
```

This will tell Rails to include a key of `relationships`, describing the related data, when serving cat records. 

So, if we define our `Cats#index` like this:

```ruby
module Api
  module V1
    class CatsController < ApplicationController

      def index
        render json: Cat.all
      end

    end
  end
end
```

We'll see this, when the client sends this request:

```bash
# GET /api/v1/cats

{
  data: [
    {
      id: "1",
      type: "cats",
      attributes: {
        name: "Moe",
        breed: "Tabby",
        weight: "fat",
        temperament: "entitled"
       },
      relationships: {
        hobbies: {
          data: [
           {
              id: "1",
              type: "hobbies"
            }
          ]
        }
      }
     },
    {
      id: "2",
      type: "cats",
      attributes: {
        name: "Ciprian",
        breed: "Calico",
        weight: "skinny",
        temperament: null
      },
      relationships: {
        hobbies: {
          data: [
            {
              id: "2",
              type: "hobbies"
            }
          ]
        }
      }
    }
  ]
}
```

We can see our `relationships` key is present, describing the associated hobby data for each cat. But we're still not side loading our data. No actual hobby records are present here. Let's fix that now. 

To side load our data, we need to add the following to our `render` call in the controller:

```ruby
render json: Cat.all, include: ['hobbies']
```

This will return the following data:

```
# GET /api/v1/cats

{
   "data": [
      {
         "id": "1",
         "type": "cats",
         "attributes": {
            "name": "Moe",
            "breed": "Tabby",
            "weight": "fat",
            "temperament": null
         },
         "relationships": {
            "hobbies": {
               "data": [
                  {
                     "id": "1",
                     "type": "hobbies"
                  }
               ]
            }
         }
      },
      {
         "id": "2",
         "type": "cats",
         "attributes": {
            "name": "Ciprian",
            "breed": "Calico",
            "weight": "skinny",
            "temperament": null
         },
         "relationships": {
            "hobbies": {
               "data": [
                  {
                     "id": "2",
                     "type": "hobbies"
                  }
               ]
            }
         }
      }
   ],
   "included": [
      {
         "id": "1",
         "type": "hobbies",
         "attributes": {
            "name": "eating"
         },
         "relationships": {
            "cats": {
               "data": [
                  {
                     "id": "1",
                     "type": "cats"
                  }
               ]
            }
         }
      },
      {
         "id": "2",
         "type": "hobbies",
         "attributes": {
            "name": "playing"
         },
         "relationships": {
            "cats": {
               "data": [
                  {
                     "id": "2",
                     "type": "cats"
                  }
               ]
            }
         }
      }
   ]
}
```

Now, we have our top level key of `included` which contains the actual hobby records that are associated to the requests cats. 

**Optimization: Avoiding N+1**

Currently, however, this data is a bit slow to load, because it is sending a request to the database for the hobbies of *each cat*, one at a time. We're executing the following two database requests:

```
Cat Load (0.4ms)  SELECT "cats".* FROM "cats" INNER JOIN "cat_hobbies" ON "cats"."id" = "cat_hobbies"."cat_id" WHERE "cat_hobbies"."hobby_id" = $1  [["hobby_id", 1]]

Cat Load (0.2ms)  SELECT "cats".* FROM "cats" INNER JOIN "cat_hobbies" ON "cats"."id" = "cat_hobbies"."cat_id" WHERE "cat_hobbies"."hobby_id" = $1  [["hobby_id", 2]]
```

Let's clean this up, and only query our database once, for *all* the hobbies associated to *all* the cats. Here's how:

Change your `render` call in the following way:

```ruby
render json: Cat.includes(:hobbies), include: ['hobbies']
```

This changes our database query to the following:

```
CatHobby Load (1.1ms)  SELECT "cat_hobbies".* FROM "cat_hobbies" WHERE "cat_hobbies"."hobby_id" IN (1, 2)
```

Much more efficient! Instead of querying the database for cat hobbies, once for each cat in our collection, we'll make one query for all the cat hobbies we need. 

**Hobby Serializer**

You may notice that the data we're serving on hobby records includes the `relationship` key, pointing to data describing related cats in turn. That's because I've already set up my Hobby Serializer. Let's take a look:

```ruby
class HobbySerializer < ActiveModel::Serializer
  attributes :id, :name
  has_many :cats
end
```

Additionally, you can use the following in your Hobby Controller to ensure that cat records are `included`, i.e. side loaded, in the response to any request for hobbies:

```ruby
module Api
  module V1
    class HobbiesController < ApplicationController

      def index
        render json: Hobby.includes(:cats), include: 
          ['cats']
      end
    end
  end
end
```

And that's it!
