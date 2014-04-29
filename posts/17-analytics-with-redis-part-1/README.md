+ [Part 2](http://codemy.net/posts/analytics-with-redis-part-2)

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