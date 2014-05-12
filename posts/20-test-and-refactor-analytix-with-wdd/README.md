In this episode we're going to be focusing on making our analytics system cleaner. We did the most straight forward things in the previous episodes and we've reached a point where we understand how our system works and how it all fits in the big picture. 

Now its time for us to test our solution and clean up the code. 

We're going to do it using a method I've become comfortable with and thats called 'Wish Driven Development'. Sounds fancy right?

## Things we need to do

+ Clean up Key Generation
+ Clean up the DSL for tracking objects

## Basic Structure

Lets start out by setting up the basic structure of our abstraction layer. We'll start by thinking about what we want to do.

Usually its good to start with what it is you 'wished' would happen, since we're crafting our own system its up to us to design our own domain-specific-language.

### I Wish...

I wish that for us to track an object all we had to do was something like this.

```ruby
@post.track :views, [:uniques, request.remote_ip]
```

Now that we know what to expect lets go ahead and make our wish come true.

## Lib

In your application's `lib` folder go ahead and create a file called `analytix.rb`.

Lets start out with defining our module

```ruby
module Analytix

end
```

We need some thing to help us generate the key in the structure that we need

```
2014:5:9:post:1:views
```

Lets breakdown what this key consists of

+ The Date
+ The class name of the object
+ The ID of the object
+ The name of the statistic we're tracking

### I Wish...

I wish that I could generate the key using something like this.

```ruby
Key.build_for(stat, object)
```

That means we need to create a module called `Key` in our `Analytix` module. With the `build_for` method being a class method

```ruby
module Analytix
  module Key
    def self.build_for(stat, object)

    end
  end
end
```

Alright before we go ahead and fill out our `build_for` method lets write the test

In `spec/lib/analytics_spec.rb` go ahead and fill this out

```ruby
require 'spec_helper'
require 'analytix'

module Analytix
  describe Key do 
    # mock object 
    let(:post_class) { double("object", { name: 'Post' })}
    let(:post)       { double(Post, { id: 1, class: post_class })}
    
    let(:date)       { Date.today }
    
    let(:views_key)  { "#{date.year}:#{date.month}:#{date.day}:post:1:views" }

  end
end
```

We need to mock the `post` object because when we generate the key we'll need to use the `post.class.name` to generate the class name of the object and `post.id` will generate the `id`.

So basically the above code sets up our testing environment so that we have everything we need to generate our key

Lets continue writing our test

```ruby
describe "#build_for" do 
  it "should create key for :views" do 
    expect(Key.build_for(:views, post)).to eq views_key
  end 
end 
```

That should get us started. Obviously the test is going to fail. Lets implement our solution

```ruby
def self.build_for(stat, object)
  date = Date.today
  [date.year, date.month, date.day, 
   object.class.name, object.id, stat.to_s].join(':')
end
```

Now your tests should pass.

## The Model

Since we expect to call the `track` method from the object we desire to track we will need to include it as a mixin into the `class` of the object.

We know that different type of stat needs to be tracked differently, for example our `:views` statistic is tracked using the `incr` method and our `:uniques` statistic is tracked with the `sadd` method. So we will need to handle that as well.

```ruby
module Analytics
  module Model
    extend ActiveSupport::Concern

    def track *stats
      
    end

    def views 
      $analytix.incr Key.build_for(:views, self)
    end

    def uniques data
      $analytix.sadd Key.build_for(:uniques, self), data
    end
  end
end
```

We have now implemented the part that makes the call and record the data to Redis however we are still missing the part that will ensure that the right method gets called in the right way.

With our `stats` splat we're going to get an array of something like this 

```ruby
[:views, [:uniques, "127.0.0.1"]]
```

Basically we're going to get just a symbol or we're going to get an array with the name of the method and the data we need.

We need some sort of method that will decide how the stat recording method is called, either with an argument or without.

We can write the test for this method. Lets call the method `track_for`

```ruby
describe "#track_for" do 
  it "should call the :views method when symbol" do
    post.should_receive(:views).once
    post.track_for :views
  end

  it "should call the :uniques method when array" do 
    post.should_receive(:uniques).with('127.0.0.1').once
    post.track_for [:uniques, '127.0.0.1']
  end
end
```

And the implementation for `track_for`

```ruby
def track_for stat
  { symbol: -> { send(stat) },
    array:  -> { send(stat[0], stat[1])}
  }.freeze[stat.class.name.underscore.to_sym].call
end
```

Lets continue with the `track` method. We're going to use redis's `pipelined` method in order to optimize the call of multiple commands in 1 connection. Which means we're going to get an array with the results inside, like so.

We know what needs to happen. When we call the `incr` method redis simply returns the number of views to us. So we can write the test like this

```ruby
  it "should call the correct method when just symbol" do 
    post.track(:views).should eq [1]
    post.track(:views).should eq [2]
  end
```

If we call the unique method we can expect something like this.

```ruby
  it "should call the correct method when just array" do 
    post.track([:uniques, "127.0.0.1"]).should eq [true]
    post.track([:uniques, "127.0.0.1"]).should eq [false]
  end
```

And finally when we pass in both stats we can expect something like this 

```ruby
  it "should return multiple results with multiple stats" do 
    post.track(:views, [:uniques, "127.0.0.1"]).should eq [1, true]
    post.track(:views, [:uniques, "127.0.0.1"]).should eq [2, false]
  end
```

Lets have a look at the final implementation for the `track` method

```ruby
def track *stats
  $analytix.pipelined do 
    stats.each { |stat| track_for(stat) }
  end
end 
```

## Final Spec File

```ruby
require 'spec_helper'
require 'analytix'

module Analytix
  describe Model do 
    let(:klass_name) { double("Object", { name: "Post" }) }
    let(:post)       { double(Post, { id: 1, class: klass_name }).extend(Model) }

    before do 
      keys = $analytix.keys('*')
      $analytix.del(keys) if keys.present?
    end

    describe "#track" do 
      it "should call the correct method when just symbol" do 
        post.track(:views).should eq [1]
        post.track(:views).should eq [2]
      end

      it "should call the correct method when just array" do 
        post.track([:uniques, "127.0.0.1"]).should eq [true]
        post.track([:uniques, "127.0.0.1"]).should eq [false]
      end

      it "should return multiple results with multiple stats" do 
        post.track(:views, [:uniques, "127.0.0.1"]).should eq [1, true]
        post.track(:views, [:uniques, "127.0.0.1"]).should eq [2, false]
      end
    end

    describe "#track_for" do 
      it "should call the :views method when symbol" do
        post.should_receive(:views).once
        post.track_for :views
      end

      it "should call the :uniques method when array" do 
        post.should_receive(:uniques).with('127.0.0.1').once
        post.track_for [:uniques, '127.0.0.1']
      end
    end
  end

  describe Key do 
    let(:klass_name) { double("Object", { name: "Post" }) }
    let(:post)       { double(Post, { id: 1, class: klass_name })}
    let(:date)       { Date.today }

    let(:views_key)  { "#{date.year}:#{date.month}:#{date.day}:post:1:views" }

    describe "#build_for" do 
      it "should create key for :views" do 
        expect(Key.build_for(:views, post)).to eq views_key
      end 
    end 
  end
end
```

## Final Implementation File

```ruby
module Analytix
  module Model
    extend ActiveSupport::Concern

    def track *stats
      $analytix.pipelined do 
        stats.each { |stat| track_for(stat) }
      end
    end

    def track_for stat
      { symbol: -> { send(stat) },
        array:  -> { send(stat[0], stat[1])}
      }.freeze[stat.class.name.underscore.to_sym].call
    end

  private

    def views
      $analytix.incr Key.build_for :views, self
    end

    def uniques data
      $analytix.sadd Key.build_for(:uniques, self), data
    end
  end

  module Key
    def self.build_for stat, object
      date = Date.today

      [date.year, date.month, date.day,
       object.class.name.underscore, object.id, stat].join(':')
    end
  end
end
```