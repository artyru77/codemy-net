+ [Part 1](http://codemy.net/posts/analytics-with-redis-part-1)
+ [Part 2](http://codemy.net/posts/analytics-with-redis-part-2)

Alright so now we understand how our analytics system is going to work we can now integrate what we've just learned into our Rails app.

Lets first figure out how we can integrate Redis into our rails app. Its actually quite simple. We're going to use an initializer to load redis when our application starts.

In `config/initializers` go ahead and create a file called `redis.rb` with the following content

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
    $redis.sadd "#{Date.today.year}:#{Date.today.month}:#{Date.today.day}:post:#{@post.id}:uniques", request.remote_ip
  end
```

Ok so thats pretty much all we need to track our views and unique visitors. 

There is a lot of code smell here and we can clean it all up but lets take a pause here and continue in the next video!