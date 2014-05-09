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
    subject { Key }
  end
end
```

We need to mock the `post` object because when we generate the key we'll need to use the `post.class.name` to generate the class name of the object and `post.id` will generate the `id`.

So basically the above code sets up our testing environment so that we have everything we need to generate our key

Lets continue writing our test

```ruby
describe "#build_for" do 
  it "should build key for :views" do 
    expect(
      subject.build_for(:views, post)
    ).to eq "#{date.year}:#{date.month}:#{date.day}:post:1:views"
  end

  it "should build key for :uniques" do 
    expect(
      subject.build_for(:views, post)
    ).to eq "#{date.year}:#{date.month}:#{date.day}:post:1:uniques"
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

Lets first write our test for the `track` method. 

We know what needs to happen. When we call the `incr` method redis simply returns the number of views to us. So we can write the test like this