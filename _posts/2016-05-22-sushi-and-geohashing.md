---
layout: post
title: "Sushi and geohashing"
language: english
category: posts
---

I have been a big fan of the sushi chain [Itsu](https://itsu.com) ever since I moved to London. Not only is the food tasty and well prepared, they also do a [half price sale 30 minutes before closing](https://www.itsu.com/about/sale/). When I was living in East London I would frequent my nearest shop at least twice a week as they were closing.

After moving further away from their shops, I tried to keep their half price sale in mind whenever I happened to be in central London in the evening. It became a bit time consuming to look nearby shops up on Google Map and check each one for its closing time, so I spent some time prototyping something that would automate the task for me.

By scraping their website and iterate through all stores, I was able to construct a simple index of branches, their address and closing hours. Coupled with [a lookup database for UK post codes to latitude/longitude](https://www.freemaptools.com/download-uk-postcode-lat-lng.htm) I built a rudimentary single page Javascript application that uses the [HTML5 geolocation API](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation/Using_geolocation) to show a list of branches sorted by distance and time remaining until the sale starts.

Recently I have spent some time refactoring the code, which [can be found here](https://github.com/nlindblad/sushi-bargain) along with [the single page web application](https://github.com/nlindblad/sushi-bargain-frontend).

### The original approach

The data format of each branch would be as follow:

{% highlight json %}
{
  "position": {
    "lat": 51.5138245139901,
    "lng": -0.075389253238543
  },
  "post_code": "EC3N1BD",
  "name": "Aldgate",
  "half_price_times": [
    null,
    20.5,
    20.5,
    20.5,
    20.5,
    20.5,
    null
  ],
  "address": "77 Aldgate High Street, London"
}
{% endhighlight %}

The Javascript application simply downloaded an array of these objects and sorted the using a custom comparator method:

{% highlight javascript %}
var sortedPossibleBranches = possibleBranches.sort(function (a, b) {
        return a.timeUntilHalfPrice - b.timeUntilHalfPrice || a.distance - b.distance;
    });
{% endhighlight %}

where `distance` is the distance from the user to the branch, calculated using the [haversine formula](https://en.wikipedia.org/wiki/Haversine_formula):

{% highlight javascript %}
function degrees2Radians(degrees) {
  return degrees * Math.PI / 180;
}

function distance(lon1, lat1, lon2, lat2) {
  var earthRadius = 6371;
  var dLat = degrees2Radians(lat2-lat1);
  var dLon = degrees2Radians(lon2-lon1);
  var a = Math.sin(dLat/2) * Math.sin(dLat/2) +
          Math.cos(degrees2Radians(lat1)) * Math.cos(degrees2Radians(lat2)) *
          Math.sin(dLon/2) * Math.sin(dLon/2);
  var c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1-a));
  var d = earthRadius * c;
  return d;
}
{% endhighlight %}

Even though the number of branches is small (68 at the moment) and this can be performed by the browser on a decent device within a few seconds, I was curious how the lookup could be made more efficient.

### Optimising the lookup

The main process of gathering the information in the first place is to scrape the Itsu website (done responsibly using [requests-cache](http://requests-cache.readthedocs.io/en/latest/)). This upfront work required to compile the data file is already quite time consuming, so adding extra steps that would improve the runtime performance on the client seemed reasonable from a time perspective.

Since the lookup is based on both the time until the half price sale and the location to the user, I was looking for ways to pre-compute as much as possible with regards to opening hours and the shops locations.

#### Time slot lookup table

In the original implementation, on each page load all stores were iterated and the time until the half price sale for the current weekday was calculated and stored:

{% highlight javascript %}
for (var i = 0; i < data.length; i++) {
  var e = data[i];
  var halfPriceTime = e.halfPriceTimes[currentDay];
  if (halfPriceTime != null) {
    e.timeUntilHalfPrice = halfPriceTime - currentTime;
    if (e.timeUntilHalfPrice >= -0.5) {
      ...
    }
  }
}
{% endhighlight %}

One optimisation that came to mind was to simply pre-populate a lookup table which partitions the week into half hour time slots. That equates to an array with `7 * 24 * 2 = 336` elements. Each element is simply an array with the index of each store whose half price sale starts at that exact half hour.

That way, by calculating the current time slot:

{% highlight javascript %}
let now = new Date;
let currentTimeSlot = now.getDay() * 24 * 2 + now.getHours() * 2;
{% endhighlight %}

we can just start at that time slot in the lookup table and iterate a few time slots ahead to find all shops with half price sale times in that specific interval, which stops us having to iterate all shops in order to know which ones have a sale on soon.

#### Introducing Geohashing

After reading up on [geohashes](https://en.wikipedia.org/wiki/Geohash) (not to be confused with [geohashing](https://xkcd.com/426/)) I decided to 
