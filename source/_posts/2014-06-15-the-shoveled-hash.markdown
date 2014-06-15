---
layout: post
title: "The Shoveled Hash"
date: 2014-06-15 18:50:54 -0400
comments: true
categories: 
---
#Where's My Shovel asks the Hash

As a typical Ruby student, I became acquainted with arrays before hashes. The array shovel, aka `<<` really made sense to me and once I moved on to Hashes I found it very disconcerting that hashes were left shovelless. 

Now that my studies have moved into classes and duck punching I decided I would try and give my hashes shovels and see what became of it.

##A hash without a shovel

If you ever attempted to use a shovel with a hash, the ruby interpreter greets you with:
```Ruby
{:a => "one", :b => "two", :c => "three"} << {:d => "three"}
=>NoMethodError: undefined method `<<' for {:a=>"one", :b=>"two", :c=>"three"}:Hash
```
In the above scenario, the main ways to add the `{:d => "three"}` to the existing hash would be to use `merge!` or `store`

`merge!` 
```Ruby
{:a => "one", :b => "two", :c => "three"}.merge!({:d => "three"})
 => {:a=>"one", :b=>"two", :c=>"three", :d=>"three"} 
 ```
`store`
 ```Ruby
test = {:a => "one", :b => "two", :c => "three"}
 => {:a=>"one", :b=>"two", :c=>"three"} 

test.store(:d, "three")
 => "three" 

test
 => {:a=>"one", :b=>"two", :c=>"three", :d=>"three"} 
```
Despite the verbosity of the above methods, they also have the affect of replacing a key's value if you `merge!` or `store` using a key that already exists in the hash. 

##A hash with a shovel is a beautiful thing 

```Ruby
{:a => "one", :b => "two"} << {:c => "three"}
=> {:a=>"one", :b=>"two", :c=>"three"}
```

What if the key you shovel already exists? Wouldn't it be nice if the `<<` method took the existing value in the hash and put it into an array and then added the passed values? I think so.

```ruby
{:a => "one", :b => "two"} << {:b => "an additional b value!!"}
=> {:a=>"one", :b=>["two", "an additional b value!!"]}

```

What if the the key exists and it has a __value that is already an array__. Wouldn't it be nice if the shoveled value was justed added to the array?
```ruby 
{:a => "one", :b => ["two", 2]} << {:b => "an additional b value!!"}
=> {:a=>"one", :b=>["two", 2, "an additional b value!!"]}
```
Tested using Ruby Version 2.12