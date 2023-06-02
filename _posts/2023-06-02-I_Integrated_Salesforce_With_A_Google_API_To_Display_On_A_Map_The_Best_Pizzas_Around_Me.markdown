---
layout: post
title: "I've integrated Salesforce With a Google API to display on a map the best pizzerias around me"
date: 2023-06-02 09:00:00 +0300
categories: jekyll update
---

<p>Hey! We have to agree. Pizza is one of the most beautiful and comforting foods in the world. Every time is an occasion to discover some new pizzerias. And to be honest, even if I am an adventurer(hmmm I am not so sure, let me check on my Trailhead profile...No, I am a Ranger now), I have to challenge myself even more, by finding pizzerias I wouldn't go to normally. Thus, I've developed this Lightning Web Component to allow me to find the best pizzerias around me. And, who knows everything about everything(apart from ChatGPT, I mean)? Yes, Google! So, for this development, we will integrate Salesforce with an API from Google called Places API. I don't want to do some advertising on their API, but I've found it very easy to use, with very well-written documentation. Now that everything was said, let's move to the integration!</p>

<h3>Step one: Integration</h3>
<p>First, we have to sign up to the <a href="https://console.cloud.google.com/welcome/">Google for developers</a>. By doing so, you get some free credit, which will be more than enough to realize our tests. You can then go to this <a href="https://console.cloud.google.com/apis/credentials">link</a>to get your credentials.</p>
<p>Concerning this integration, according to the documentation, the only listed way to access the API is through a key, which is given on your Google parameters. And you still have to be cautious about this: according to the Salesforce documentation, custom metadata types can store protected information (like API keys), but you have to set the custom metadata type as "Protected". You have to remember this, it's very important. Also, the drawback we have by using a remote site setting instead of a named credential is that if the key has to change for one reason or another, we can't connect anymore(until we set the new value on our custom metadata type record).</p>
![Custom Metadata Type Object](/Images/Google_Places_Integration_mdt_object.jpg)
<br>
![Custom Metadata Type Record](/Images/Google_Places_Integration_mdt_record.jpg)
<br>
<p>When it's done, we go to the Remote site settings, and we authorize the endpoint we are going to use. Here the authorization is done from the endpoint, so the most straightforward way to do it is through a remote site setting. If we had to use an OAuth authentication, for example, we would have used a named credential for this.</p>
<br>
![Remote Site Setting](/Images/Google_Places_Integration_Remote_Site_Settings.jpg)

<p>And now, let's make the magic happen!</p>

<h3>Step 2: The Apex part</h3>
<p>On this development, there is definitely where the magic happens. Apex is doing everything here. Because let me talk about what we want exactly. We want exactly this: </p>
{% highlight javascript %}
mapMarkers = [
    {
        location: {
            Street: '1 Market St',
            City: 'San Francisco',
            Country: 'USA',
        },
        title: 'The Landmark Building',
        description:
            'Historic <b>11-story</b> building completed in <i>1916</i>',
    },
];
{% endhighlight %}
<p>This development doesn't need a lot of customization to work. If you look at some resources on the internet, you will see that a lot of them are just about filling this list in the right format(in the JavaScript file), and that's it. Concerning our work, after the callout, we only select the data we want, modify it when it's needed, and set it in the right format. For example, when we transform the value of the price range(1, 2, 3, 4) to a dollar symbol. Or, when we add some (very light) HTML code to the description field, to display multiple elements there. By the way, I've struggled a lot trying to add a "stars system" on my description field, or even on the mapMarkers list. For now, it's not possible. The advantage of the lightning map is that you don't have to work a lot to make it work. But the drawback is that you can't do whatever you want, it's hardly customizable. So yes, you can add some HTML, but that's it. So, no nested LWC, no images to display inside it, just some paragraphs and texts. If you try, the tags will simply be removed. But maybe on the next release we will be able to do more with the lightning-map components!  </p>
{% highlight java %}
public with sharing class googleCalloutHandler {
  //When you get the result from the API, you get a 1, 2, 3, or 4 for its price range.
  //I didn't find it very meaningful, so I've changed it to a "dollar" symbol. Everybody understands Dollars!
  public static String getReadablePriceLevel(Integer googlePriceLevel) {
    switch on (googlePriceLevel) {
      when 1 {
        return '$';
      }
      when 2 {
        return '$$';
      }
      when 3 {
        return '$$$';
      }
      when 4 {
        return '$$$$';
      }
      when else {
        return '';
      }
    }
  }
  //Same here: I find "Yes"/"No" a more understandable answer than "True"/"False"
  public static String isOpen(Boolean googleIsOpen) {
    if (googleIsOpen == true) {
      return 'Yes';
    } else {
      return 'No';
    }
  }
  //We don't get a proper address from this API, but we get an address at least.
  //So, instead of making a new callout to another API(and this for EACH result, which would take a huge amount of resources),
  //I've just chosen to transform the given address to make it usable by our lightning-map
  public static Map<String, String> getDisplayableAddress(String givenAddress) {
    Pattern regex = Pattern.compile('^(.*?),([^,]+)$'); //We define our regex pattern
    Matcher matcher = regex.matcher(givenAddress); //We apply the regex to our string
    Map<String, String> addressComponents = new Map<String, String>(); //We define a map to store the street name and the city
    if (matcher.find() && matcher.groupCount() >= 2) {
      //If we find at least two results(street and city), we isolate the two groups, and we save them to the map
      String lastExpressionBeforeComma = matcher.group(1).trim();
      String lastExpressionAfterComma = matcher.group(2).trim();
      addressComponents.put('street', lastExpressionBeforeComma);
      addressComponents.put('city', lastExpressionAfterComma);
      return addressComponents;
    }
    return null; // If we didn't find results, we return null
  }
  //We call this method from a wire, so the cacheable=true is mandatory here
  @AuraEnabled(cacheable=true)
  public static List<Map<String, Object>> getRestaurants(
    String latitude,
    String longitude
  ) {
    //We define the request. We want some data from Google, we already have everything to make the callout(by everything, I am talking about the token.
    //We don't need to make a 'POST' callout, to receive a token, that you will reuse for your second callout. You already have it from Google
    Http http = new Http();
    HttpRequest Request = new HttpRequest();
    Request.setMethod('GET');
    String radius = '500'; //The radius is the distance(in meters) between the pizzeria and the user.
    //By doing this, if a pizza hut is 1500 meters from the user's computer, it won't be returned by Google API
    String type = 'restaurant'; //It's another filter, to get some accurate data. We won't get a travel agency(but we can ask for them if we want)
    //The key will be reused on the link. We get it from a custom metadata type we created before, and we add it to the endpoint.
    String key = String.valueOf(
      API_Credentials__mdt.getInstance('Google_Maps_API').get('Token__c')
    );
    //Good practice here: instead of using a "+", "+" string concatenation,
    //I've used the String.format method, which is allowing us to add new  //dynamic elements
    String url = String.format(
      'https://maps.googleapis.com/maps/api/place/nearbysearch/json?keyword=pizza&location={0}%2C{1}&radius={2}&type={3}&key={4}',
      new List<string>{ latitude, longitude, radius, type, key }
    );

    //When the url is ready, we can define the endpoint of our request, and send it!
    Request.setEndpoint(url);
    HttpResponse Response = http.send(Request);
    //Note here: the successful status code is 200. It has to be checked by doing some tests,
    //it can change from web services to another(with the Twilio integration it was 201)
    if (Response.getStatusCode() == 200) {
      //When the callout is successful, we create the mapMarkers list on the createMapMarkersFromResponse(with the response's body
      //as a parameter), and we return it to the LWC.
      return googleCalloutHandler.createMapMarkersFromResponse(
        Response.getBody()
      );
    }
    return null; //If we have another status, we got null. It's exactly like doing an if...else here
  }

  public static List<Map<String, Object>> createMapMarkersFromResponse(
    String res
  ) {
    //These variables will be used to fill the mapMarkers list
    String name;
    String price;
    String rating;
    String vicinity;
    String user_ratings_total;
    String isOpen;
    //mapMarkers is a list of maps. It has the same format of the list we have on the documentation(and it has to be, otherwise it won't work)
    List<Map<String, Object>> mapMarkers = new List<Map<String, Object>>();
    //We get a string. We deserialize it to get a map
    Map<String, Object> data = (Map<String, Object>) JSON.deserializeUntyped(
      res
    );
    //We got a "list of results" on the JSON, so we convert the object to a list of objects.
    //By doing this, we can increment inside the results data list, and get the elements we want
    List<Object> results = (List<Object>) data.get('results');
    for (Object result : results) {
      Map<String, Object> place = (Map<String, Object>) result;
      name = String.valueOf(place.get('name'));
      rating = String.valueOf(place.get('rating'));
      price = getReadablePriceLevel(Integer.valueOf(place.get('price_level')));
      Map<String, Object> openNode = (Map<String, Object>) place.get(
        'opening_hours'
      );
      //Some elements are still unknown to Google and can provoke some null pointer exceptions.
      //I added a non null condition and a safe operator("?.") to avoid it,
      //but normally i should verify the null condition(and the data accuracy) for each element i want to display
      Object googleIsOpen = openNode?.get('open_now');
      if (googleIsOpen != null) {
        isOpen = isOpen(Boolean.valueOf(googleIsOpen));
      }

      vicinity = String.ValueOf(place.get('vicinity')); //Vicinity is the "address" given by Google. I didn't change the variable's name
      user_ratings_total = String.ValueOf(place.get('user_ratings_total')); //We get the number of ratings, in addition to the average rating
      Map<String, Object> mapMarker = new Map<String, Object>();
      Map<String, Object> markerLocation = new Map<String, Object>();
      //We transform the address to be able to display it on the lightning-map element
      Map<String, String> addressMap = googleCalloutHandler.getDisplayableAddress(
        vicinity
      );
      markerLocation.put('Street', addressMap.get('street'));
      markerLocation.put('City', addressMap.get('city'));
      mapMarker.put('location', markerLocation);
      mapMarker.put('value', name);
      mapMarker.put('title', name);
      //Important here: On the description, we can add some HTML tags, but not everything. Basically, we can just add some paragraphs, titles, or bold characters.
      //I will send the link to the documentation on my article.
      //We also used a String.format method, as we did previously.
      mapMarker.put(
        'description',
        String.format(
          '<p><b>Rating:</b> {0}({1} persons voted)</p><p><b>Price Level:</b> {2}</p><p><b>Open Now:</b> {3}</p>',
          new List<String>{ rating, user_ratings_total, price, isOpen }
        )
      );
      //We add a new pizzeria mapMarker to the list
      mapMarkers.add(mapMarker);
    }
    //When it's done, we return the final list to the getRestaurants method
    return mapMarkers;
  }
}
{% endhighlight %}

<h3>JavaScript</h3>
<p>Let's think as a final user. What do we want? We want that, when the user opens the page(it could be any page, I've chosen the Home Page, but it has no importance), we have to collect its geolocation. And when it's done, we can fetch the data we want from Apex. So, let's go! We defined for this a getUserLocation method, called when the page loads. When it's done, the variables latitude and longitude will change. And fortunately, the <a href="https://developer.salesforce.com/docs/component-library/documentation/en/lwc/lwc.data_wire_service_about">wire</a> service on Lightning Web Components is called every time it parameters change!</p>

{% highlight javascript %}
import { LightningElement, track, wire } from "lwc"; //The wire import will be used to get the data, and the "track" will allow us to get reactive values of the mapMarkers list
import getRestaurants from "@salesforce/apex/googleCalloutHandler.getRestaurants";
import { getLocationService } from "lightning/mobileCapabilities";

export default class RestaurantsNearMeMap extends LightningElement {
  //Every pizzeria we get from Google is an element of the mapMarkers list
  @track mapMarkers = [];
  //Latitude and longitude will store the current position of the user. They will be useful to call the Google API
  latitude;
  longitude;
  // We use a getter for this one. We ask our LWC to center the map around the actual geolocation of the user
  get center() {
    return {
      location: { Latitude: this.latitude, Longitude: this.longitude }
    };
  }
  //The "$" is important. It allows us to handle the case when our variables are null. If we don't, we will get an error message when the LWC will render
  @wire(getRestaurants, {
    latitude: "$latitude",
    longitude: "$longitude"
  })
  wiredAccount({ error, data }) {
    //We added an "if...else here because it's better to handle errors when they happen. But it's not mandatory, just a better practice
    //To be displayed, the mapMarker has to have a specific format, with specific fields. To be sure that every element is right,
    //We've handled the mapMarker in one only place: the apex class.
    //By doing this, we just have to give the returned data from Apex to the JS mapMarkers list.
    if (data) {
      this.mapMarkers = data;
      this.error = undefined;
    } else if (error) {
      this.error = error;
      this.mapMarkers = undefined;
    }
  }
  //When the page loads(ie "when the element is inserted into a document"), we call getUserLocation
  connectedCallback() {
    this.getUserLocation();
  }
  //We get the actual geolocation of the current user
  getUserLocation() {
    //By doing this, we check if the browser currently supports the geolocation functionality.
    //If yes, we can save the user position on the latitude and longitude variables
    if (navigator.geolocation) {
      navigator.geolocation.getCurrentPosition((position) => {
        this.latitude = position.coords.latitude;
        this.longitude = position.coords.longitude;
      });
    }
  }
}
{% endhighlight %}

<h3>The HTML Part</h3>
<p>To be honest, I've even hesitated to give you the HTML file as it was really empty. But it's showing you that here, we almost don't use HTML. The component is already in its finished state. So, yes, we can change few parameters, like how to center the map, or the size of the zoom, but basically, that's it.</p>
{% highlight html %}
<template>
  <lightning-card title="Best pizzerias around me">
    <lightning-map
      map-markers={mapMarkers}
      markers-title="Best pizzerias around me"
      center={center}
      zoom-level="16"
    >
    </lightning-map>
  </lightning-card>
</template>
{% endhighlight %}

<h3>Results</h3>
<p>When you've added everything on your LWC including your meta file, you can see this result. I know that this app is totally a life changer, because it will allow you to know exactly what(pizza) and where to eat. </p>

![Result](/Images/Google_Places_Integration_result.jpg)
<br>
<h3>Sources</h3>
<ul>
  <li><a href="https://developer.salesforce.com/docs/component-library/bundle/lightning-map/documentation">Lightning Map documentation</a></li>
  <li><a href="https://developers.google.com/maps/documentation/places/web-service/overview?hl=en">Google Places API documentation</a></li>
  <li><a href="https://github.com/selimhamidou/pizzeriasNearMe">Github repo</a></li>
</ul>




