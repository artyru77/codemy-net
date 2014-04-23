Ruby's true strength is its syntax. You can write really beautiful code using ruby. Thats the reason why most people fall in love with it. One of the things ruby isn't good at is performance. Its an interpreted language that runs on a VM so its much slower than compiled languages. 

When making web requests we have to be very careful. We have to ensure that we can get the data we need with as less computations and i/o hit as possible. We want to put all the number crunching outside of the request, either via loading pre computed data from the cache or asynchronously calculate the result in a separate process. The request should just be a matter of fetching the answer that is pre computed or if we need to do any computation it should be minimal and efficient.

The most key thing one can do while developing ruby on rails application is to practice BDD (benchmark driven development). There are many ways to solve a problem and its up to you as a developer to find the most elegant solution to the problem. You have to work with what you have. For example we know that **CPU** time is expensive, and **RAM** and **DISK** are cheaper. Your solutions should be to use as much of the cheap resource as possible and least of the expensive stuff. We have to strike a balance between performance / cost. Get as much out of our hardware as we possibly can.

Imagine this scenario. We need to sort our users into categories based on their age. Anyone below 10 years old will be classified as a 'child', anyone between 10 and 18 years old will be 'teenager' and anyone between 18 and 24 years of age would be considered as 'young', between 25 to 54 is considered 'adult', anyone with 

```ruby
if age < 10 then 'child'
  elsif age < 18 then 'teenager'
  elsif age < 24 then 'young'
  elsif age < 54 then 'adult'
  ...
  else 'senior'
end
```

This code is fine, its even fast. If statements are always naturally fast. In this case our computation is very simple however the code can be difficult to read if it gets more complex, imagine if you had more than 4-5 categories for different age groups.

A cleaner way to implement this code would be to use switches.

```ruby
case age
  when 0..9 then 'child'
  when 10..18 then 'teenager'
  when 19..28 then 'young'
  when 29..50 then 'adult'
  when 50..100 then 'senior'
  else 'deity'
end
```

This piece of code is a bit more readable that the if statement solution. However case / switch statements in ruby are pretty slow. 

So there should be a way where we can get good performance and readable code. What if we had a hash of all the combinations of all the results. Could we perhaps just do a hash look up? Yes we could! Lets take a look at it.

```ruby
AGE_CATEGORIES = { (0..9) => :child, (10..18) => :teenager, 
                   (19..28) => :young, (29..50) => :adult, 
                   (50..120) => :senoir }.freeze
```

Now all we have to do is look up the category we want for a given age.

```ruby
AGE_CATEGORIES.detect { |k, v| k.include?(age) }.last
```

### How I tested 

```ruby
10000000.times do 
  user = User.new(Random.rand(100))
  user.if_category # method that calls code that determines the category
end
```

Basically we created the user instance with the age and call the methods that does the decision making for the category using the different methods 10,000,000 times.

```
                      user     system      total        real
if                5.540000   0.010000   5.550000 (  5.541052)
case             12.590000   0.000000  12.590000 ( 12.600189)
lookup           29.780000   0.080000  29.860000 ( 29.859172)
```

With the lookup its actually even slower than the case / switch. Even though our code is cleaner we've lost performance. The reason why its slower is because its not just doing a lookup its actually checking to see if the number is in the range via ```k.include?(age)```. We've eliminated code but increased computational requirement.

## There has got to be a better way

The slowest part of the lookup is checking whether the **age** is in the range of the key of the hash. So what can we do to eliminate this computation. Well we can just simply multiply out all the possibility, since we have a finite number of possibilities. So we want something like this.

```ruby
{ 1 => :child, 2 => :child, 3 => :child } # and so on...
```

Some of you might argue that its tedious to write out. Well  we don't need to write it out manually we can use what we have to generate that data and cache it when our app starts. Check this out.

```ruby
AGE_CATEGORIES = { (0..9) => :child, (10..18) => :teenager, 
                   (19..28) => :young, (29..50) => :adult, 
                   (50..120) => :senoir }.freeze

COMPUTED_CATEGORIES = Hash[*AGE_CATEGORIES.keys
                       .map { |r| r.map { |v| 
                         [v, AGE_CATEGORIES.select 
                           { |k, val| 
                             k.include?(v)
                           }.values.first]
                         } }.flatten].freeze
```

You may be thinking this code is mega ugly, and it is. So we can actually clean this up. 

```ruby
class AgeGroup < Struct.new(:categories)
  def calculate_table
    Hash[*categories.keys.map{ |r| expand_range(r) }.flatten]
  end

  def expand_range(range)
    range.map { |n| make_age_array(n) }
  end

  def make_age_array(age)
    [age, find_match_for(age)]
  end

  def find_match_for(age)
    categories.detect { |k, val| k.include?(age) }.last
  end
end
```

We can use a service object to clean up the code that generates the hash. Now all we have to do is just call this from our class. Here is the User class in its entirety

```ruby
class User < Struct.new(:age)
  AGE_CATEGORIES = { (0..9) => :child, (10..18) => :teenager, 
                     (19..28) => :young, (29..50) => :adult, 
                     (50..120) => :senoir }.freeze

  COMPUTED_CATEGORIES = AgeGroup.new(AGE_CATEGORIES).calculate_table

  def lookup_cached
    COMPUTED_CATEGORIES[age]
  end
end
```

Ok now we have our solution how does it perform? Well lets see! 

```
                      user     system      total        real
if                5.890000   0.010000   5.900000 (  5.901544)
case             12.880000   0.000000  12.880000 ( 12.883877)
lookup           29.930000   0.050000  29.980000 ( 29.984650)
lookup_cached     4.520000   0.000000   4.520000 (  4.521481)
```

Things are looking pretty good for our cached lookup method. Its basically a 26% improvement in speed over the if statement solution. 

You might think this seems like over engineering. But the point of this article is to let you know there are other possibilities other than using if statements and case / switch. When we're building our system we have to think about how we can optimise our code. Eliminate any computation that is not needed. If you can save 20ms here and 50ms there why not?

This is a very simple example. Obviously in a real use case you would have to look at your particular case and figure out the best way to go about your solution. Maybe a hash lookup might not work. The fact is every system has different problems. The point I am trying to make here is, don't just stay in your comfort zone. Sometimes the best solution requires you to think outside the box. I've recently had to engineer a solution for a hotel availability system that is based on dates and seasonal pricing. It virtually has an infinite possibility but I was able to find a viable solution for it using some of the techniques I mentioned here, although with a slightly different execution. All in all we were able to shave off an average of between 200ms - 400ms off the response time.

If you would like to see the entire benchmark file here it is [user_category.rb](https://gist.github.com/zacksiri/6199461)

Update 1: Changed from using ```select``` to ```detect``` suggested by [@scomma](http://twitter.com/scomma) and [@hasclass](http://twitter.com/hasclass)

new results

```
                      user     system      total        real
if                5.310000   0.010000   5.320000 (  5.312411)
case             13.010000   0.000000  13.010000 ( 13.008580)
lookup           22.370000   0.010000  22.380000 ( 22.372320)
lookup_cached     4.340000   0.010000   4.350000 (  4.342154)
```