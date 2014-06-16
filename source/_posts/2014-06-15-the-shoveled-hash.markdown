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
```ruby
{:a => "one", :b => "two", :c => "three"} << {:d => "three"}
=>NoMethodError: undefined method `<<' for {:a=>"one", :b=>"two", :c=>"three"}:Hash
```
In the above scenario, the main ways to add the `{:d => "three"}` to the existing hash would be to use `merge!` or `store`

`merge!` 
```ruby
{:a => "one", :b => "two", :c => "three"}.merge!({:d => "three"})
 => {:a=>"one", :b=>"two", :c=>"three", :d=>"three"} 
 ```
`store`
 ```ruby
test = {:a => "one", :b => "two", :c => "three"}
 => {:a=>"one", :b=>"two", :c=>"three"} 

test.store(:d, "three")
 => "three" 

test
 => {:a=>"one", :b=>"two", :c=>"three", :d=>"three"} 
```
Despite the verbosity of the above methods, they also have the affect of replacing a key's value if you `merge!` or `store` using a key that already exists in the hash. 

##A hash with a shovel is a beautiful thing 

```ruby
{:a => "one", :b => "two"} << {:c => "three"}
=> {:a=>"one", :b=>"two", :c=>"three"}
```

The above seems so simple and elegant. Taking it further, what if the key you shovel already exists? Wouldn't it be nice if the `<<` method took the existing value in the hash and put it into an array and then added the passed values? I think so.

```ruby
{:a => "one", :b => "two"} << {:b => "an additional b value!!"}
=> {:a=>"one", :b=>["two", "an additional b value!!"]}

```

What if the the key exists and it has a __value that is already an array__? Wouldn't it be nice if the shoveled value was justed added to the array?
```ruby 
{:a => "one", :b => ["two", 2]} << {:b => "an additional b value!!"}
=> {:a=>"one", :b=>["two", 2, "an additional b value!!"]}
```

###Making the above a reality - duck punching the Hash class
Achieving the above can be done by adding to / duck punching the Hash class. Defining a `class Hash` at the top of your ruby script is the simplest way to achieve this. The ruby interpretor will read from this version of the Hash class before looking up the main Hash class included with ruby. Any new methods added will exist in addition to the original Hash class methods. 

A very simple implementation
```ruby
class Hash
  def <<(a)
    self.merge!(a)
  end
end
``` 

With this simple class method added to Hash class you can now shovel an existing hash with another hash passed in the `{}` format. 
The ruby interpreter allows you to do `hash1 << hash2` whereas most other methods would require the `.`, ie `hash1 << hash2` is the same as hash1.<< hash2. In the above the (a) is the passed parameter given after the `<<` 

```ruby
{:a => "alpha"} << {:b => "bravo"}
 => {:a=>"alpha", :b=>"bravo"} 
```
###Done with the baby steps
While the above is great, the << method needs to grow a bit to give me the full functionality I described above (plus more!)
Below is my full Hash class duck punch, followed by some explanations. 

```ruby
class Hash
  def <<(a, b=nil)
    if a.class == Hash
      passed_key = a.first[0]
      passed_value = a.first[1]
    end
    if a.class == Array
      passed_key = a[0]
      passed_value = a[1..-1] if a.length > 2 #more than 2 values in passed array means pass_value becomes an array of all the values after [0]
      passed_value = a[1] if a.length == 2 # only 2 values in array so first must be key and 2nd must be value
    end
    if !b.nil? #if a second argument is passed, treat it as first being the key and second being the value
      passed_key=a
      passed_value = b
    end
    
    #IMPLIMENTATION 
    if self[passed_key].nil?
      self.merge!({passed_key => passed_value})
    else #if the key passed exists, then take current value of key and put it in array w/ passed valued
      temp = self[passed_key] #grab current value of existing key
        if temp.class == Array #if the existing value is already an array then add to that array
          self[passed_key] << passed_value
        else
          self[passed_key] = [temp,passed_value] #create an array with original value and passed value
      end
    end
    
    return self
  end
  
  def self.array 
    Hash.new {|hash, key| hash[key] = []} 
  end 

end
```

I added an optional parameter `b` which allows you to pass 2 parameters, the first being the key and the second being the value. Unfortunately, ruby does not let you use the space `<< ` shortcut explained above when passing two parameters. Instead, the only way to pass this second parameter (as far as I know) is using the `hash1.<< key,value`. 
```ruby
{:a => "one", :b => "two"}.<<:c,"three"
=> {:a=>"one", :b=>"two", :c=>"three"}
```

The rest of this new `<<` method expects only one parameter, thus the space `<<` space can be used.

The initial if statements determine the class type of the parameter passed and break it into a key and a value assigned to `passed_key` and `passed_value` respectively. 

The array method conditional allows you to pass a single parameter as an array. The first element of the array will be treated as the key and the subsequent ones will be values. If there are multiple values passed they are kept in array.
```ruby
{:a => "one", :b => "two"} << [:c, "three"]
=> {:a=>"one", :b=>"two", :c=>"three"}
```

The first step under #IMPLIMENTATION is to see if the hash that is being added to already has a key with the same name as the key bing passed. When accessing a non existent hash key, the ruby interpreter will return nil. Thus if `self[passed_key].nil?` equals true, then the key does not exist in the original hash and a simple merge! can be used. 


If the key does exist, then the existing value is put into an array and the passed value is added to that array. If the existing value is already an array then the passed value is added to the array. 

```ruby
 {:a => "one", :b => "two", :c => ["three", "additional value"]} << {:c => "a third value"}
=> {:a=>"one", :b=>"two", :c=>["three", "additional value", "a third value"]}
```

####The final bit
```ruby
def self.array 
    Hash.new {|hash, key| hash[key] = []} 
end 
```
This method is a little different from the main topic of the post. However, while I was ducking punching the hash class I figured I'd add in a simple but very useful functionality that also relates to the `<<`. This functionality is based on fellow classmate [Peter's blog post](http://pcrglennon.github.io/blog/2014/06/10/rubys-hash-dot-new-and-%7B%7D/) regarding initializing a hash with empty array values. 

Instead of having to type out `a = Hash.new {|hash, key| hash[key] = []}` I can just type `a = Hash.array`. Both of these initialize an empty hash with a default value of an empty array. This allows you to  add to the hash by specifying a new or existing key and using a shovel to add to the array. 
```ruby
a = Hash.array
=> {}
a[:new_key] << "Value to be added to value array" 
=> ["Value to be added to value array"]

a
=> {:new_key=>["Value to be added to value array"]}
```

##Practicality
It is admittedly not convenient to add this lengthy duck punch at the top of your scripts. You could put this code in a separate file and `include` it, but not only does every person running your ruby code also need the class modification, doing this may create confusion since it is not part of the normal ruby Hash class. 

With that said, this new method does add functionality that comports with the style already used for `<<` and arrays, thus giving a code reader contextual clues that strongly hint at the methods functionality. Additionally, it is my hope that this experiment provides a basis for the consideration and discussion of possibly seeing how a shovel could be introduced as an official method to the Hash class.  

*Tested using Ruby Version 2.1.2*