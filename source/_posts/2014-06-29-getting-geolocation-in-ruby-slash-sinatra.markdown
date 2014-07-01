---
layout: post
title: "Getting Geolocation in Ruby / Sinatra"
date: 2014-06-29 17:44:34 -0400
comments: true
categories: 
---

#Getting Location in Ruby
It is now very common to determine a user's location by using HTML5 Geolocation. Below, I'll discuss how to pass this geolocation information to a Ruby Sinatra environment by using the post method and some JavaScript. 

While the HTML5 geolocation is great, its not available if you are only operating in a command line interface (CLI). Thus, I'd like to first point out a way to get location when your ruby program is command line only. 

##Location in CLI Only Environment 
This is a very simple way to get surprisingly accurate (at least in my few tests in NYC) location information using the IP of the computer running the Ruby app, Nokogiri web scraping, and one's IP address.

You will need gem 'nokogiri' and to require 'open-uri'
```ruby

def location
  page = "http://freegeoip.net/json/"
  doc = Nokogiri::HTML(open(page, 'User-Agent' => 'ruby'))
  loc = /(latitude)(\":)(\d+.\d+)(,\"longitude\":)(-\d+.\d+)/.match(doc.text)
  lat = loc[3]
  lon = loc[5]
  return [lat,lon]
end
```

This will query the site http://freegeoip.net/json which in turn determines your public IP address and matches it against a database. Nokogiri then scrapes this result and then I use a Ruby regex match object to parse out the latitude and a longitude. Finally I return an array containing latidue and longitude.

I found that http://freegeoip.net/json was fairly accurate. There is a gem called geocoder that seems to use the same database but it seemed to do a lot more and required active record. When I created this, I already was using nokogiri in the project so this was an easy add on and just seemed simpler.

##Location using HTML5 in a Sinatra environment
This describes how to use HTML5 Geolocation and javascript to send your longitude / latitude to the Sintra server using a http post request.

###Overview
It starts by loading a page that says getting location and requests the user's location from their browser. Once it has a location, it loads a google map on the page showing it. While loading the map, javascript also fills in 2 hidden form values, one for latitude and the other longitude. 

The user then clicks the Submit Location button which will do a post request to the server at the address specified in the form's action attribute. Sinatra can then grab lat and long from the parmas variable and render a new page using erb or preform a redirect. 

I'll note that there may be a better way to do the following but this method seemed good enough and was within the scope of my current knowledge. 

####Step 1
This methods assume you are running a Ruby Sintra web based frame work. 

First, you would set a `get` controller for your page that will request the location from the user's browser using html. I used by root index page for this. 
```ruby 
get '/' do
    erb :location
end
```
####Step 2
Next, you set up an ERB template that includes a hidden form. Once the user's browser responds to the geolocation request, this hidden form will be filled by JavaScript with the user's longitude and latitude.

```java
<!DOCTYPE html>
<html>
<body>

<p id="demo">Getting your location... </p>
<div id="mapholder"></div>

<script src="http://maps.google.com/maps/api/js?sensor=false"></script>

<script>
getLocation();
var x = document.getElementById("demo");

function getLocation() {
    if (navigator.geolocation) {
        navigator.geolocation.getCurrentPosition(showPosition,showError);
    } else { 
        x.innerHTML = "Geolocation is not supported by this browser.";}
}

function showPosition(position) {

    lat = position.coords.latitude;
    lon = position.coords.longitude;
    document.getElementById('lat').value = lat;
    document.getElementById('lon').value = lon;
 
    latlon = new google.maps.LatLng(lat, lon)
    mapholder = document.getElementById('mapholder')
    mapholder.style.height='250px';
    mapholder.style.width='500px';

    var myOptions={
    center:latlon,zoom:14,
    mapTypeId:google.maps.MapTypeId.ROADMAP,
    mapTypeControl:false,
    navigationControlOptions:{style:google.maps.NavigationControlStyle.SMALL}
    }
    
    var map = new google.maps.Map(document.getElementById("mapholder"),myOptions);
    var marker = new google.maps.Marker({position:latlon,map:map,title:"You are here!"});

    
}

function showError(error) {
    switch(error.code) {
        case error.PERMISSION_DENIED:
            x.innerHTML = "User denied the request for Geolocation."
            break;
        case error.POSITION_UNAVAILABLE:
            x.innerHTML = "Location information is unavailable."
            break;
        case error.TIMEOUT:
            x.innerHTML = "The request to get user location timed out."
            break;
        case error.UNKNOWN_ERROR:
            x.innerHTML = "An unknown error occurred."
            break;
    }
    
}
</script>
<form id="myForm" action='/go' method="post">
  <input type='hidden' id="lat" name='lat' value='' />
  <input type='hidden' id="lon" name='lon' value='' />
  <input type='submit' name='submit' value='Submit Location' />
</form>

</body>
</html>
``` 
In the `showPosition(position)` function above the following JavaScript code causes the hidden form at the bottom to be filled with the user's longitude and latitude values. 
```java
document.getElementById('lat').value = lat;
document.getElementById('lon').value = lon;
```
####Step 3
Setup up a post controller that receives the post request from the form submission. 

```ruby
post '/go' do
    @lat = params[:lat]
    @lon = params[:lon]
    erb :index  

end
```
The parmas var is automatically passed to the post method in Sinatra  and you can then extract the lat and lon by calling the symbol version of their names as they were labled in the form. You can then pass the location info to another method or make them directly available to the erb template.  

### Using both HTML5 Geolocation with an IP based fallback
You can use both the methods described above, reserving the IP based geolocation for situations in which HTML5 Geolocation fails. While you could probably do this with Javascript and error handling, I achieved this by passing the location to a Ruby class method. By checking if the lat or lon paramaters were nil, I was able to conditionally fall back to the IP based lookup.

```ruby
def self.set_location (lat,lon)
    if lat.nil?|| lon.nil? #if html5 location isn't passed, revert to this method
      page = "http://freegeoip.net/json/"
      doc = Nokogiri::HTML(open(page, 'User-Agent' => 'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_9_3) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/35.0.1916.153 Safari/537.36'))
      loc = /(latitude)(\":)(\d+.\d+)(,\"longitude\":)(-\d+.\d+)/.match(doc.text)
      @lat = loc[3]
      @lon = loc[5]
    else
      @lat = lat
      @lon = lon
    end
```

*HTML5 Geolocation code based on http://www.w3schools.com/ templates*