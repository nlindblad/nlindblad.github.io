---
layout: post
title: "Sushi and geohashing"
language: english
category: posts
---

I have been a big fan of the sushi chain [Itsu](https://itsu.com) ever since I moved to London. Not only is the food tasty and well prepared, they also do a [half price sale 30 minutes before closing](https://www.itsu.com/about/sale/). When I was living in East London I would frequent my nearest shop at least twice a week as they were closing.

After moving further away from their shops, I tried to keep their half price sale in mind whenever I happened to be in central London in the evening. It became a bit time consuming to look nearby shops up on Google Map and check each one for its closing time, so I spent half an hour prototyping something that would automate the task for me.

By scraping their website and iterate through all stores, I was able to construct a simple index of branches, their address and closing hours. Coupled with [a lookup database for UK post codes to latitude/longitude](https://www.freemaptools.com/download-uk-postcode-lat-lng.htm) I built a rudimentary single page Javascript application that uses the [HTML5 geolocation API](https://developer.mozilla.org/en-US/docs/Web/API/Geolocation/Using_geolocation) to show a list of branches sorted by distance and time remaining until the sale starts.

Recently I have spent some time refactoring the code, which [can be found here](https://github.com/nlindblad/sushi-bargain).

### The original geolocation search

The data format of each branch would be as follow:

```
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
```

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

### Introducing geohashing

After reading up on [geohashing](https://en.wikipedia.org/wiki/Geohash) I decided to
