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
Our user table simply stored the active recorded created **user_id**, **created_at**, **updated_at**, as well as **name**, **provider**, **uid**, and **image**. The provider column represented the provider used by omniauth, which in our case was always Twitter. The uid column was the unique id returned by the oath provider, twitter in our case, and the image was the Twitter profile image of the user. 

A city_users join table kept the user city relationships. 


Our site relies heavily on javascript and ajax calls so that it never has to load another page after the initial page load. Thus, our controllers would pass JSON objects, containing the cities objects, info on a users logged in status, uid's of saved items, ect.., back to our front end javascript. 

##Adding temporary anonymous users
Searching Google for creating users just based on a temporary basis using omniauth didn't yield any results so we set off to figure it out on our own. 

We settled on using the `session_id` that rails automatically creates and stores in a cookie for any visitor on your site. This id string is unique for any given visitor and thus we used this session_id as the unique identifier to store in the user's uid column. 

The first step in implementing this logically fell with our cities controller which handled the task of adding new cities and associating them with the current user. (Technically probably best to move this functionality into the model) When a new city was passed to the cities controller Create method via an ajax post request, the first thing the method did was set an @user variable:

```ruby
@user = User.find(session[:user_id]) if session[:user_id] 
```

The `session[:user_id]` was a variable the sessions controller would add to the session cookie after a successful twitter oauth negotiation, aka only logged in users would have it. This variable also corresponds to the `user_id` assigned to a user when it is created with active record. 

The first step was to create the temp user when the controller received a new city and the current user was not logged in. 

```ruby
    if !@user #create a temp user tied to session ID
      @user = User.create(name: 'Guest', provider: 'anon', uid: session[:session_id], image: 'https://origin.ih.constantcontact.com/fs197/1110193228531/img/301.jpg?a=1115291249439')
      @user.cities << [City.find(1), City.find(2), City.find(3), City.find(4), City.find(5)]
      session[:user_id] = @user.id 
    end

    unless @user.cities.find_by(lat: city_params[:lat], lon: city_params[:lon])
     #above checks if the user already has this city and rejects the addition if so
      @user.cities << @city if @user
    
     #the below handles deleting a city, which is mostly beyond the scope of this post. 
      del_city_id = city_params[:lastClock].to_i
    
      CityUser.find_by(user_id: @user.id, city_id: del_city_id).destroy if @user
      render json: @city
    else
      render json: "this city already exists".to_json
    end
```

The user would immediately get shoveled in the 5 default cities, allowing them to be treated as a normal user that should always have exactly 5 cities. Whenever a new city is added, the last city is deleted and the new city is added as the first. 

Also note we set provider to anon. This allowed us to know this was a temporary user account. We also added to this temp user's session cookie the user_id variable that now corresponded to their  user_id in our users' table. 

Overall this allowed us to treat temporary users that could be treated like a real user without having to change much code. 

##When a temp user isn't a real user
There were some distinctions that we decided would be practical to keep. We didn't want our temp user's to be able to save content from within each of their cities and these temp users still needed the option to actually log in with twitter. 

These distinctions actually fit within our code base pretty well. Basically we wanted to treat a temp user as not logged in except for purposes of tracking which cities they had. 

Rails was keeping track of whether or not a user was logged in via a helper method in the application controller called logged_in?

```ruby
helper_method def logged_in?
    !!current_user 
end

helper_method def current_user
    @current_user ||= User.find(session[:user_id]) if session[:user_id]
end
```

We added a simple conditional check that made it so `logged_in?` would only return true if the provider did not equal anon in addition to their being a user_id var in the session.

```ruby
helper_method def logged_in?
    !!current_user && @current_user.provider != "anon"
  end
```
On the front end side we created a similar global variable called `loggedIn` that would get set and update as needed via ajax requests. The rails users controller sets the value of this variable based on checking the users logged in status. 

This meant we had a temp user whose cities would persist, but for all other purposes they would be treated as logged out. 

##Transitioning a temp user to a twitter authenticated user
To make the user experience as seamless as possible we wanted to make it so that a temp user who customized their cities and then logged in to our site with their twitter account for the first time would be able to keep their cities. 

To do this we modified the Sessions Controller's Create method which we had originally made when implementing OmniAuth with Twitter. This route is called as the twitter oauth callback and thus is invoked right after someone authenticates with twitter.

We added the third nested `if` that checks to see if the session contains a `user_id`. This was a perfect test for finding temp users who then authenticate with twitter because only a temp user or a fully logged in user would have a session user_id; however, a user who is already logged in would never be routed to this create method, an oauth callback route. 

```ruby
def create
  if auth_hash.present?
    if !@user = User.find_by_provider_and_uid(auth_hash[:provider], auth_hash[:uid]) #ie a new twitter auth user. 
      if session[:user_id] #ie if was a temp user & this is first time authing with twitter then replace their temp account uid, provider, name and image with this new auth hash info. 
        @user = User.find(session[:user_id])
        @user.uid = auth_hash[:uid]
        @user.provider = auth_hash[:provider]
        @user.name = auth_hash[:info][:name]
        @user.image = auth_hash[:info][:image]
        @user.save
      else
        @user = User.create_from_omniauth(auth_hash)
        @user.cities << [City.find(1), City.find(2), City.find(3), City.find(4), City.find(5)]
      end
    end

    if @user
      session[:user_id] = @user.id
      redirect_to root_url
    else
      redirect_to root_url
    end
  else
    redirect_to root_url, :notice => "Please login via twitter"
  end
end
```
This third nested `if` sets the @user variable to the temp user if it finds a session `user_id`. It then replaces the `uid` of that user with the twitter provided oauth uid and also replaces the `provider`, `name` and `image` provided by the twitter authentication. Effectively this converts the temp user to a twitter authenticated, persistent world-view user who retains the cities they selected as a temp user.

If a user has authenticated with twitter on our site before but they come back as an anonymous temp user, when they re-authenticate with twitter, their information from their temp account is not passed on to their previous user.  

#A database full of temp users
One slight draw back to our temp user approach is that our database fills up with temp users who will never be accessed after a temp users session cookie is lost or expires. 

On solution to this is to create a cron job that periodically checks for users with provider anon who were created over 30 days ago and then purges them. 


