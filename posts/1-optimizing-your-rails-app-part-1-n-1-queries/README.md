This post is old and has been deprecated. We have created videos that will help guide you to solving the N+1 problem in Rails. Check it out over here

## [Free N+1 Video Series](http://codemy.net/posts/search/playlists/a-shot-of-ruby/tags/n-1/order/asc)

+ [Introduction to N+1 Problem](http://codemy.net/posts/introduction-to-n-1-problem)
+ [Optimized Count with Joins and Custom Select](http://codemy.net/posts/optimized-count-with-joins-and-custom-select)
+ [Eager Loading with Active Record](http://codemy.net/posts/eager-loading-with-active-record)


> There is a lot of talk about ruby on rails and its performance, and how it doesn't scale. Fact of the matter is rails does scale. There are many apps built using rails and are processing millions of request / day. Basecamp, Harvest, Shopify are some examples of large scale rails app that serve 100s of millions request / day so why do people still complain about its performance?
>
> In this series of articles we take a look at some of the most common pitfalls that causes rails application to have these performance issues, and how to fix them. In this part we're going to be talking mostly about N + 1 queries.
>
> ## Database Queries
>
> First of all we have to understand that an average request in a web application is very likely going to make on an average about 5 - 6 database queries. Thats considered a good request.
>
> In a lot of cases 1 request could even be making 10 - 20 database queries. This means that web applications are usually I/O bound. Your going to mostly be waiting for the database to serve the queries. In a well written rails application making on average 5-6 requests for a GET request should be the goal. For a POST or a PUT even 10-20 is acceptable however not ideal. Remember less is always more in any form of computing.
>
> ### Avoiding N+1 queries
>
> I've come across many rails application during my teaching days and this seems to be a mistake people who are new to rails makes a lot. 
>
> What are N + 1 queries? Its basically when you load a bunch of objects from the database and then for each one of those objects you make 1 or more database query, let me explain with an example.
>
> ```ruby
> # your models
> 
> class Customer < ActiveRecord::Base
>   has_one :address
    scope :active, -> { where(active: true) }
> end
>
> class Address < ActiveRecord::Base
>   belongs_to :customer
> end
>
> ```
>
>
> ```ruby
> # your controller
>
> def index
>  @customers = Customer.active
> end
> ```
>
> ```ruby
> # your view
>
> <% @customers.each do |c| %>
  <%= c.name %> # this is fine the object is already in memory from your controller
  <%= c.address %> # this one makes a query to the database
<% end %>
```
>
> So as your outputting the address for each customer your going to make a query to the database, imagine loading up 100s of customers in your index and having to make 100s of queries to the database. Your really going to slow your application down.
>
> ### Say hello to eager loading
> 
> To avoid N + 1 queries you can use a technique called eager loading to load the address of the customers at the same time the customers are being loaded from the database. So everything happens in 1 big query. How do you do this in rails? Lets take a look
> 
>```ruby
>  @customers = Customer.active.includes(:address)
> ```
>
> By using includes your telling ActiveRecord to load the address along with the customer, and ensuring everything happens in 1 query. You can reduce 100 queries into 1 - 2 queries using this technique. If you know your going to use the data from the address in one way or another you might as well just eager load it into memory for that particular request.
>
> You can also do eager loading in your model, where you setup your relationships.
>
> Check out this example
>
> ```ruby
> class Customer < ActiveRecord::Base
>   has_many :orders, -> { includes(:items) }
> end
>
> class Order < ActiveRecord::Base
>   has_many :items
> end
>
> class Items < ActiveRecord::Base
>   belongs_to :order
>   belongs_to :product
> end
> ```
>
> When you make a call to the orders of the customer for example
>
> ```ruby
> customer.orders
> ```
>
> ActiveRecord is going to eager load all the items associated with the orders as well so you can avoid N + 1 this is good if you want to then do things like loop out each order and have a count of how many items each order has.
> 
> ```ruby
> <% customer.orders.each do |order| %>
>   <%= order.number %> (<%= order.items.count %>)
> <% end %>
> ```
>
> If you ever feel your rails app is slow one of the first things you should look for are these N + 1 queries. How do you spot them? Simply look at your development logs you'll see an exceptionally long list of database queries getting put out by rails.
>
> Eager loading is one of those things that new comers to rails usually miss, because in a development environment there is 1 user making the request to the database, and everything is happening locally. If one user is making 100 request to the database on a dedicated hardware (computers you develop with are usually more powerful than production servers) its not going to make much of a difference. However imagine this issue in production, 500 users making N + 1 queries at the same time. You can imagine how the computing power and memory on production server getting saturated very quickly.