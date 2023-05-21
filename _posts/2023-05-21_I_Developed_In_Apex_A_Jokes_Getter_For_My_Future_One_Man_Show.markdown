---
layout: post
title: "I developed in Apex a jokes getter for my future one man show"
date: 2023-05-21 08:49:26 +0300
categories: jekyll update
---

<p>Hey! As a full time joker, I love to laugh. This development was, as the beginning, just a way to begin my days with something funny. Generally, daily meetings are at 10AM, so my goal was to receive each day a new joke I would remember about. And, at the middle of my development, I said: "Oh, why wouldn't I lanch my one man show? I can get all the jokes I want, WITH NO EFFORT AT ALL! 
So, from this moment, the Salesforce notifications feed became a manager which is still ungry at 10AM, still giving me some jokes, but putting pressure on me like: "here is my jokes, you better be funny this time". 
I automated a little bit more the process, by saving the jokes on a new object, and displaying the total number of them on the feed. But that's basically what my Salesforce development does: getting jokes, saving them, and displaying them on the screen. 
So, here is the story behind this article.</p>

<h3>Step one: Configuration</h3>
<p>Now that you know everything about the context, let's move to the practical part. 
To get the jokes, no need to scrape all the internet to get them. I used the <a href="https://api-ninjas.com/api/jokes">Joke API</a> from API Ninja. It allows us up to 50000 callout per month, so it's more than enough for our daily usage. To use it, you just have to sign up, and to go to <a href="https://api-ninjas.com/profile">your profile</a> to see your token. Keep this tab open, we will go back to this.</p>
![Joke API Token](/Images/Joke_API_Token.jpg)

<p>Now, we have to store this token.To do this, we create a custom metadata type called Joke_API_Integration__mdt with a custom field named Token__c. When the custom metadata type and the field are created, you can copy and paste the key on it.</p>
![Joke API Custom Metadata Type Fields](/Images/Joke_API_Mdt_Fields.jpg)
![Joke API Custom Metadata Type Records](/Images/Joke_API_Mdt_Record.jpg)
<p>The next thing to do is to authorize the connexion between Salesforce and the API. For this, we add a Remote Site Setting.</p>
![Joke API Remote Site Settings](/Images/Joke_API_Remote_Site_Settings.jpg)

<p>And when it's done, we create a custom object called Joke_For_My_One_Man_Show__c, with the Joke__c field. it will be used to store our jokes, in the case we would like to read them again.</p>
![Joke API Object Manager](/Images/Joke_API_Object_Manager.jpg)
![Joke API Object Manager Field](/Images/Joke_API_Object_Manager_Field.jpg)

<p>And now, let's move to the the notifications part. How to notify the user that a new joke is coming? We could use platform events, but it's not necessary. We also could use show toast events, but the problem is that the message would only show for a specific object, and not in the entire org. For our need, custom notifications is the perfect choice. By using them, we would allow users to see them, no matter they are in Salesforce. Now that we know this, we also have to know that a custom notification needs a type to exist. We create it by going to Setup->Custom Notifications->New. 
When it's done, we can move to the Step 2.
![Joke API Custom Notification Type](/Images/Joke_API_Custom_Notification_Type.jpg)

<h3>Step 2: The Apex part</h3>
<p>To be honest, I've reused the skeleton of the code of the <a href="https://www.selimhamidou.com/posts/I_Developed_A_Solution_To_Receive_SMS_Alert_Before_A_Meeting">Twilio Integration I did yesterday</a>, because the notions involved are the same: we have to schedule a REST callout to an API, so we use the schedulable interface with a future method. As we will see later, some details are different between the two integrations, but there are still a lot of similarities.</p>

{% highlight java %}
public with sharing class GetJokesFromAPIHandler implements Schedulable {
  @future(callout=true)
  public static void callAPI() {
    //For this callout we only need the token. We've stored it inside a Joke_API_Integration__mdt custom metadata type
    String key = String.valueOf(
      Joke_API_Integration__mdt.getInstance('API_Token').get('Token__c')
    );
    //We define the number of jokes we want from the callout. For us it's 1, but it could be changed
    Integer NumberOfResults = 1;

    //We define a HTTP request
    Http http = new Http();
    HttpRequest request = new HttpRequest();

    //We define the endpoint. It will take as a parameter the number of results
    request.setEndpoint(
      'https://api.api-ninjas.com/v1/jokes?limit=' + NumberOfResults
    );
    //We add the token inside the header
    request.setHeader('X-Api-Key', key);

    //We don't have jokes right now, we need one. We use a 'GET' method
    request.setMethod('GET');

    //We send the request
    HttpResponse response = http.send(request);

    // If the request is successful, we parse the JSON response
    if (response.getStatusCode() == 200) {
      //We got this kind of response: [{"joke": "My Joke"}]. So, we need to transform it to only get "My joke" as a String
      List<Object> input = (List<Object>) JSON.deserializeUntyped(
        response.getBody()
      );
      String joke = (String) ((Map<String, Object>) input.get(0)).get('joke');
      //When it's done, we call sendJokeNotification, which will create a custom notification
      sendJokeNotification(joke);
    } else {
      system.debug('failure');
    }
  }
  //We use the execute method of the Schedulable interface to call the Joke API
  public void execute(SchedulableContext SC) {
    callAPI();
  }
  public static void sendJokeNotification(String joke) {
    //We get a list of the
    User userWhoLikesToLaugh = [
      SELECT Id
      FROM User
      WHERE Name = 'Sélim Hamidou'
      LIMIT 1
    ];

    // We get the notification type Id
    CustomNotificationType notificationType = [
      SELECT Id, DeveloperName
      FROM CustomNotificationType
      WHERE DeveloperName = 'Joke_Notification'
    ];

    // Create a new custom notification
    Messaging.CustomNotification notification = new Messaging.CustomNotification();

    // We call insertNewJokeForMyOneManShow, and get the number of records inside the Joke_For_My_One_Man_Show__c object
    Integer numberOfJokesForTheOneManShow = insertNewJokeForMyOneManShow(joke);
    //We add the joke to the notifications
    notification.setTitle(
      'Hey buddy, this is the joke N°' +
      numberOfJokesForTheOneManShow +
      ' for our show tonight. Don\'t disappoint me.'
    );
    notification.setBody(joke);

    // We set the notification type Id with the value we got from the previous SOQL
    notification.setNotificationTypeId(notificationType.Id);
    //We define the redirection link. We have to choose between the setTargetPageRef and the setTargetId method.
    //We choosed the first one, because we prefer to redirect to a list view instead of a specific record, but it's up to you
    //But you have to know that setting a value is mandatory here
    notification.setTargetPageRef(
      '{"type": "standard__objectPage","attributes": {"objectApiName": "Joke_For_My_One_Man_Show__c","actionName": "list"},"state":{"filterName":"00B0900000R6wKNEAZ"}}'
    );

    // When everything is set, we send the notification. If we cannot, we catch the error and display it on the debug logs
    try {
      notification.send(new Set<String>{ userWhoLikesToLaugh.Id });
    } catch (Exception e) {
      System.debug('Problem sending notification: ' + e.getMessage());
    }
  }
  //On this method we receive the joke from the API
  //We save it on a record
  //We count the number of records we have on the Joke_For_My_One_Man_Show__c object(that we created sooner), and we will reuse it on the notification body
  public static Integer insertNewJokeForMyOneManShow(String joke) {
    List<Joke_For_My_One_Man_Show__c> listOfNewJokes = new List<Joke_For_My_One_Man_Show__c>();
    Joke_For_My_One_Man_Show__c newJoke = new Joke_For_My_One_Man_Show__c(
      Name = 'Joke ' + Datetime.Now(),
      Joke__c = joke
    );
    listOfNewJokes.add(newJoke);
    insert listOfNewJokes;
    return [SELECT COUNT() FROM Joke_For_My_One_Man_Show__c];
  }
}
{% endhighlight %}

<h3>Result</h3>
<p>Now that everything is in place, you can move to your developer console, open the Anonymous Mode, and write these lines of code. It will ask our code to run everyday at 10AM.</p>
{%highlight java %}
GetJokesFromAPIHandler m = new GetJokesFromAPIHandler();
String sch = '0 0 10 * * ? *';
String jobID = System.schedule('Get New Jokes Job', sch, m);
{% endhighlight %}

And, everyday at 10AM you will receive these notifications:
![Joke API API Result](/Images/Joke_API_Result.jpg)

