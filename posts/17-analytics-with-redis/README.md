I've had a chance to use Redis in multiple ways. From basic Queue systems to more complex data caching systems.

One of the great benefits of redis is that it's a datastore that writes to memory and asynchronously writes to disk. Which means you can write lots of things into it in a very small amount of time.

Another benefit of redis is it has certain data structures baked into it. Which makes it very handy to use for multitude of things.

In this episode I'm going to be showing you how you leverage Redis datastore to build a simple analytics system for your own project. We're going to start simple with basic key-value store and then move over to use redis's data structure to store data.

## What data are we recording?

Here are the specs of the system we're going to create.

+ Track page views for a post in our blog
+ Track the unique visitors for a post
+ Track the duration the user is on the page
+ Data tracking by day

We'll start off with these basic features. Before we go any further lets figure out what it is we're going to store in redis. Page view is something which is very simple for us to track. We're its simply a counter that counts up. We don't care if its unique or not. So basically here we're just going to have a number and keep incrementing it.

For the unique visitors things get a bit more complicated. But not that much more. Lets think about it. What is the data that we can record that is unique to every visitor on our page. The IP address, right? So basically all we have to do is put a bunch of IP addresses in a list and make sure we don't duplicate it. 

Tracking the duration is probably going to be the most involving one. We know when the user lands on the page, so we can basically do a timestamp of when the user landed on the page. But how do we track when he / she leaves? Well we'll probably need some Javascript magic to help with that.

All that data is going to be sorted into days.

## Basic Redis

Before we go any further lets first understand the basics of Redis. How do we interface with redis and how we can store data into it.

### Installing Redis

```bash
brew install redis
redis-server # this starts redis
```

```bash
gem install redis
irb
```

```ruby
require 'redis'

redis = Redis.new
```

That should get you up and running with redis. Once you create the redis object you can use that instance to interface with the redis server

### Strings

Lets write something to redis

```ruby
redis.set "hello", "world"
```

In this case we're just writing some string into redis. Redis is a key value store so in this example the key is "hello" and the value is "world"

### Lists

Redis isn't just a simple key / value store. It has some data structure built in, for example a list. Lets try writing to a list.

```ruby
redis.lpush 'fruits', 'apple'
redis.lpush 'fruits', 'orange'
```

Now we have 2 items in our 'fruits' list. You might be thinking the `l` in `lpush` stands for `list` but its actually not. The `l` in this case means `left` we're pushing items into the list from the left side. Which means we can also do an `rpush`. The `rpush` command allows us to push items into the right hand side of the list. Lets try

```ruby
redis.rpush 'fruits', 'durian'
redis.rpush 'fruits', 'mango'
```

Alright now that we know how to write stuff to redis lets put it in the context of what we're trying to build. 

## Analytics System

The first thing we need to try and do is increment the pageviews. We don't care if its repeated we just want to know for each post in our blog, how many pageview it has on a day to day basis. Lets try and create that in redis.

### Tracking Page Views

Fortunately for us Redis has a command called `INCR` which just keeps track of an integer and increases the number everytime its called. Lets try it out

```ruby
redis.incr 'post:1'
```

Try running the above command a few times you'll see that it keeps track of the number and simply increments it. There is also a `decr` command take a guess what it does?

Alright in terms of tracking page views we're pretty much almost there. The only thing we need to do is sort it out into days right? Well thats very easy.

In redis when you use the `:` your namespacing your key. So essentially we can namespace our keys into days yes? 

Lets try it.

```ruby
redis.incr '2014:04:27:post:1'
```

So thats pretty simple. Right? Well how are we going to change the date. Well in Ruby we can workout a the date using something like `Date.today` or `Time.now` so when we're generating the key all we have to do is generate the key using one of those ways. We'll come back to key generation later.

What we have now is a basic data structure that will allow us to store our data in a clean manner. We can use the same kind of key to store the other analytics data we need like the unique users. But that means we have to now change our key slightly instead of just calling it `2014:04:27:post:1` we need to suffix it with what it is. So in this case we're storing the `views` so our key should be `2014:04:27:post:1:views`

### Unique Visitors

Earlier we had a look at lists however with lists we can push the same value in to the list which means its difficult for us to track 'unique' visitors using lists. Lucky for us redis has another data structure that allows you to keep track of unique items its called 'sets'

Lets try it out

```ruby
redis.sadd '2014:04:27:post:1:unique', '123.456.789.000'
```

Try adding the same IP into the set, you'll see that it returns `false` which means its not going to push the same IP into the same set. So its basically perfect for what we're trying to do.

## Putting it Together

Alright so now we understand how our analytics system is going to work we can now integrate what we've just learned into our Rails app.

Lets first figure out how we can integrate Redis into our rails app. Its actually quite simple. We're going to use an initializer to load redis when our application starts.

In `config/initializers` go ahead and create a file called `analytics.rb` with the following content

```ruby
require 'redis'

$redis = Redis.new
```

Simple right? We're creating a redis instance and saving it to a global variable for our rails app so we can access it anytime we want. (there is a better way of doing this but for the sake of keeping things simple lets go with this)

Essentially when a user visits our post he's going to go to the `posts#show` controller / action so thats probably where we need to run our code to increment our page view and track our unique visitors right? 

So basically here's what our code would look like 

```ruby
  def show
    @post = get_post
    $redis.incr "#{Date.today.year}:#{Date.today.month}:#{Date.today.day}:post:#{@post.id}:views"
  end
```

Probably not the prettiest code I've written but that would probably work. We'll worry about cleaning that up later.

The next think we need to track is the Unique visits.

```ruby
  def show
    @post = get_post
    $redis.incr "#{Date.today.year}:#{Date.today.month}:#{Date.today.day}:post:#{@post.id}:views"
    $redis.sadd "#{Date.today.year}:#{Date.today.month}:#{Date.today.day}:post:#{@post.id}:uniques", "#{request.remote_ip}"
  end
```

Ok so thats pretty much all we need to track our views and unique visitors. 

There is a lot of code smell here and we can clean it all up but lets take a pause here and continue in the next video!





