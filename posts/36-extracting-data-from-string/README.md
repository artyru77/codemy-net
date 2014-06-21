We show you how to extract the required data from a name. 

In this case we assume that the full name that gets passed in comes can be separated with a space.

For example 

```ruby
"Zack Siri".split(' ')
```

Will give us an array that looks something like this

```ruby
["Zack", "Siri"]
```

We then extract the first character from each of the item in the array.

To get the first element in the array we can use the `Array#first`

```ruby
["Zack", "Siri"].first # and
["Zack", "Siri"][0]
```

Both of the above will return `"Zack"`

We can then get ther first character from the string

```ruby
"Zack"[0] # this will give us "Z"
```