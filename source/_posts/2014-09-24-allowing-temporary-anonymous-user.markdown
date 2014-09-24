---
layout: post
title: "Allowing Temporary Anonymous Users"
date: 2014-09-24 17:33:34 -0400
comments: true
categories: 
---
#Mixing in Temporary Anonymous Users with Omniauth
On a site I made with some fellow students, world-view.today, we allowed users who logged in with Twitter to customize the site by adding in their own cities. The log in was created using omniauth 2 and basically followed the steps outlined in fellow classmate [Randall's blog](http://randallreedjr.com/blog/2014/07/15/logging-in-with-github-using-omniauth-in-your-rails-app/), substituting twitter for github. 

One of the improvements we really wanted to make was to allow non logged in users to add in cities, a task that was not as daunting as we anticipated. 

##Our site before anonymous users
Our cities controller would associate a new city with a user whenever a logged in user added a city.
Our user table simply stored the active recorded created **user_id**, **name**, **created_at**, **updated_at**, **provider**, **uid**, and **image**. The provider column represented the provider used by omniauth, which in our case was always Twitter. The uid column was the unique id returned by the oath provider, twitter in our case and the image was the Twitter profile image of the user. 

A city_users join table kept the user city relationships. 


Our site relies heavily on javascript and ajax calls so that it never has to load another page after the initial page load. Thus, our controllers would pass JSON objects back to our front end which contained the cities to show as well as the users logged in status, uid's of saved items, ect.. 

##Adding temporary anonymous users
Searching Google for creating users just based on a temporary basis using omniauth didn't yield any results so we set off to figure it out on our own. 

We settled on using the session_id that rails automatically creates and stores in a cookie for any visitor on your site. This id string is unique for any given visitor and thus we used this session_id as the unique identifier to store in the user's uid column. 

The first step in implementing this logically fell with our cities controller which handled the task of adding new cities and associating them with the current user. When a new city was passed to the cities controller create method via an ajax post request, the first thing the method did was set an @user variable:

```ruby
@user = User.find(session[:user_id]) if session[:user_id] 
``

The session[:user_id] was a variable the sessions controller would add to the session cookie after a successful twitter oauth negotiation, aka only logged in users would have it. This variable also corresponds to the user_id assigned to a user when it is created with active record. 

The first step was to create the temp user when the controller received a new city and the current user was not logged in. 

```ruby
    if !@user #create a temp user tied to session ID
      @user = User.create(name: 'Guest', provider: 'anon', uid: session[:session_id], image: 'https://origin.ih.constantcontact.com/fs197/1110193228531/img/301.jpg?a=1115291249439')
      @user.cities << [City.find(1), City.find(2), City.find(3), City.find(4), City.find(5)]
      session[:user_id] = @user.id 
    end
```

The user would get shuffled in the 5 default cities, allowing them to be treated as a normal user that should always have exactly 5 cities. Whenever a new city is added, the last city is deleted and the new city is added as the first. (So just below the above code, they are treated as a normal user and their last city is deleted and the new one added)

Also note we set provider to anon. This allowed us to allows know this was a temporary user account. We also added to this temp user's session cookie the :user_id variable that now corresponded to their temp users ID in our users table. 

Overall this allowed us to treat temporary users the same as real users without having to change much code. 

##When a temp user isn't a real user
There where some distinctions that we decided would be practical to keep. We didn't want our temp user's to be able to save content from within each of their cities and these temp users still needed the option to actually log in with twitter.

 Furthermore, it would be preferable if a temp user who had already customized their cities 

