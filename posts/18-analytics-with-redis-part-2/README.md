+ [Part 1](http://codemy.net/posts/analytics-with-redis-part-1)

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