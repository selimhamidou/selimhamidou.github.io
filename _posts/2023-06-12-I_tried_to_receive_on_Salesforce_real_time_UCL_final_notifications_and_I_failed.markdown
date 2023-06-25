---
layout: post
title: "I tried to receive Salesforce real-time notifications about the UEFA Champions League Final, and I failed..."
date: 2023-06-11 09:00:00 +0300
categories: jekyll update
---
![Mourinho](/Images/Football_API_Mourinho.jpg)
*I personally can't see any link between this image and the article, but I find it fun. Oh, maybe the link is that Mourinho cannot hear the real-time notifications, as for me on this development?*
<br>
<p>Hi! What's up? This post is really special because I've failed to implement it. You know that the last Saturday was the day of the UCL Final. Manchester City had to win it for the first time in their history, and Inter Milan had the chance to win their first Champions since 2010. So, I was imagining my evening like this: I would be watching the match(I mean, really watching), and from time to time, I would go to Salesforce, to verify if I had received some notifications about a new goal. But it didn't go well...So, I consider this development as a failure, and I would be glad if it turned out differently, but that's how life is, I think. So, I am not very proud of this development, because I could have done so much better, but I still would like to show it to you.</p>
<p>It didn't go well because I didn't test it well. Football APIs are pretty expensive. The free version of the API I used was only allowing 100 callouts a day. So, I only tested it once, on Friday night, and I considered that the response's shape would be significantly the same, which wasn't true. I was thinking I was ready, but I was not.</p>
<p>My errors led me to encounter some errors, like "NullPointerExceptions" or "List index out of bounds" errors, or even by giving the wrong endpoint while trying to call the API for a specific match. Errors are very common where you are a developer, but they wouldn't have been a problem if I did proper testing. So, I can't stress you enough about testing and retesting your code, I guess that Murphy's law was right.</p>
<br>
![Murphy's Law](/Images/Football_API_Murphy_Law.jpg)
<br>
<p>Now, let's move to the integration. I've used Apex from the beginning to the end for this integration. I wanted some real-time notifications, so I've thought about using webhooks, but unfortunately, they weren't provided by the API I've used. I wouldn't be surprised if tomorrow you contacted me on Linkedin, to tell me "Oh yes, webhooks exist for some football APIs, like THIS ONE". Honestly, I would love to hear from you, because it would make this idea prettier than it is right now. But, for now, I've simply followed the official documentation, which is telling us that updates are done every 15 seconds on the API and that it is advised to call the API every minute to get "real-time" data. We will be satisfied with that for now. And as the timing was pretty tight, I am seeing so much to do to improve my code. Again, DON'T HESITATE to reach me. I would be happy to talk about it.</p>
<h3>Step one: The API</h3>
<p> In the beginning, I was hesitating between two APIs. Finally, I've used <a href="https://www.api-football.com/">API-Football</a>, because it was giving access to lesser-known leagues for free(at the time I was doing this development, all the European leagues had finished their season, the only match to play was our match). Also, the free version only allows for 100 requests a day, but by thinking again about it, I could have used a paid version for this development, it's still cheaper than some other APIs. So, I've subscribed to the free version from <a href="https://rapidapi.com/api-sports/api/api-football-beta">Rapid API</a>. I've used the beta version of the API because this one wasn't asking me to enter my personal credit card information, instead of the proper version.</p>
<p>After being a subscriber, I've scraped the official documentation, to know what exactly to do to get the information I needed(ie the scored goals). I've finally found out that a match was a "fixture" according to this API, and that every "fixture" had an id. I still don't know when the fixture is updated, but I've considered getting it at the hour of the match. So, my plan was to make a first callout to get the fixture id, to save it to my org, and then to make some other callouts(every 3 minutes), to know if someone scored. If yes, I would have sent a notification to the current user(me). If not, nothing would happen. And my question was, how to know if a new goal has been scored? I mean, by doing exactly this, we would receive a notification every 3 minutes after the first goal of the match has been scored. Because the code will, every 3 minutes, scrape the JSON, find a goal, and then say "Great, there is a goal!". To avoid this, I've decided to store the data on a Match__c record.</p>
<br>
![Match__c object](/Images/Football_API_Match_Object_Fields.jpg)
<br>
<p>So, at the beginning of the match, I would create a record storing the fixture id, and the number of goals scored(which is 0 at this time). Before the final, my code was looking for the goals node, assembling the "home" and "away" nodes values into a string, and saving them inside my Match__c record. Unfortunately, as I said, this node was absent during the match. So instead, I created another field, numberOfGoalsScored__c, of type Number, and I was counting the number of goals instead of directly getting it from the API.</p>
<p>Now that my plan was set, I've taken my credential and copied it to a custom metadata type. This API doesn't have OAUth according to the documentation, but I can be wrong. Doesn't hesitate to correct me. In all cases, while using custom metadata types to store credentials, it's necessary to set the custom metadata types records as "Protected".</p>
<br>
![Custom Metadata Type Record](/Images/Football_API_Custom_Metadata_Type_Record.jpg)
<br>
<p>When it's done, we go to the Remote site settings and create one.</p>
<br>
![Remote Site Setting](/Images/Football_API_Remote_Site_Settings.jpg)
<br>
<p>To finish the configuration part, I have also created a custom notification type, as I did for my previous article about the <a href="https://www.selimhamidou.com/posts/I_Developed_In_Apex_A_Jokes_Getter_For_My_Future_One_Man_Show">jokes getter</a>.</p>
<br>
![Custom Notification Type](/Images/Football_API_Custom_Notification.jpg)
<br>
<h3>Step 2: The Apex code</h3>
<p>You have to know here that you can't directly schedule a job to run every three minutes. The minimum time to set between two jobs is one hour. So, to avoid this(I had to launch a job every 3 minutes, I couldn't wait for so long to get the score), I used nested jobs: I call once a first schedulable job, which will call itself 3 minutes later. And again, 3 minutes later. And so on.</p>
{% highlight java %}
public with sharing class footballAPIHandler implements Schedulable {
  //The handleCallouts method is in charge of all the callouts. We call it from a (nested) schedulable job
  public void execute(SchedulableContext SC) {
    //We don't want the callouts to be launched after the end of the match. In my time zone, the match will finish at around 11PM
    if (Datetime.now().hour() < 23) {
      //We call the API
      handleCallouts();
      //We prepare a cron trigger to launch a new job, which will launch a new job 3 minutes later, which will launch a new job 3 minutes later...
      footballAPIHandler footballAPIJob = new footballAPIHandler();
      Datetime dt = datetime.now().addMinutes(3);
      String month = String.valueOf(dt.month());
      String day = String.valueOf(dt.day());
      String hour = String.valueOf(dt.hour());
      String minutes = String.valueOf(dt.minute());
      String cronStr =
        '0 ' +
        minutes +
        ' ' +
        hour +
        ' ' +
        day +
        ' ' +
        month +
        ' ?';
      System.schedule(
        'Nested job' +
        '-' +
        System.currentTimeMillis(),
        cronStr,
        footballAPIJob
      );
    }
  }
  public void finish(Database.BatchableContext BC) {
    //Our first job has to be launched ONCE only today, not tomorrow. So we cancel it on the finish method.
    AsyncApexJob a = [
      SELECT Id, Status, NumberOfErrors, JobItemsProcessed, TotalJobItems
      FROM AsyncApexJob
      WHERE Id = :BC.getJobId()
    ];
    system.abortJob(a.id);
  }
  //We make a callout from a schedulable Apex, so we need to add a @future(callout=true)
  @future(callout=true)
  public static void handleCallouts() {
    //We get the credential from a custom metadata type record we saved
    String key = String.valueOf(
      API_Credentials__mdt.getInstance('Football_API').get('Token__c')
    );
    Http http = new Http();
    HttpRequest request = new HttpRequest();

    //These parameters are defined on the API doc
    request.setHeader('X-Rapidapi-Key', key);
    request.setHeader('X-Rapidapi-Host', 'api-football-beta.p.rapidapi.com');

    //We use a GET method to get the match information
    request.setMethod('GET');
    //We verify if we already have a record called 'Manchester City UCL Final'. We've created it on our first callout
    List<Match__c> matchList = [
      SELECT Id, score__c, API_Id__c, numberOfGoals__c
      FROM Match__c
      WHERE name = 'Manchester City UCL Final'
    ];
    //If we don't, we use the live=all parameter on our endpoint, end then, we send our request
    if (matchList.size() == 0) {
      request.setEndpoint(
        'https://api-football-beta.p.rapidapi.com/fixtures?live=all'
      );
      HttpResponse httpresponse = http.send(request);
      //if the request has succeeded, we handle this callout on the firstCallout method
      if (httpresponse.getStatusCode() == 200) {
        firstCallout(httpresponse.getBody());
      }
    //if we do, we can call the API with the fixture ID, that we stored on the Match__c record
    } else {
      request.setEndpoint(
        'https://api-football-beta.p.rapidapi.com/fixtures/events?fixture=' +
        String.valueOf(matchList[0].API_Id__c)
      );
      HttpResponse httpresponse = http.send(request);
      //if the request has succeeded, we call the otherCallouts() method
      if (httpresponse.getStatusCode() == 200) {
        otherCallouts(matchList[0], httpresponse.getBody());
      }
    }
  }
  //The only job of this method is to create a Match__c record for the current match, with the data we get from the API
  public static void firstCallout(String jsonResponse) {
    //We transform our String response to a map
    Map<String, Object> responseMap = (Map<String, Object>) JSON.deserializeUntyped(
      jsonResponse
    );
    List<Object> responseList = (List<Object>) (responseMap.get('response'));
    List<Match__c> matchToQuery = new List<Match__c>();
    //We get multiple matches, so multiple responses
    for (Object eachResponse : responseList) {
      Map<String, Object> teamsMap = (Map<String, Object>) eachResponse;
      //We verify for each team if its name contains 'manchester'
      Map<String, Object> teams = (Map<String, Object>) teamsMap.get('teams');
      for (String eachTeam : teams.keySet()) {
        Map<String, Object> mappedTeam = (Map<String, Object>) teams.get(
          eachTeam
        );
        if (
          String.valueOf(mappedTeam.get('name'))
            .toLowerCase()
            .contains('manchester')
        ) {
          //if yes, we get its fixture id, and save it to a new Match__c record, called 'Manchester City UCL Final'
          Object fixtureObj = ((Map<String, Object>) eachResponse)
            .get('fixture');
          Map<String, Object> fixtureObjMap = (Map<String, Object>) fixtureObj;
          String fixtureId = String.valueOf(fixtureObjMap.get('id'));
          matchToQuery.add(
            new Match__c(
              name = 'Manchester City UCL Final',
              API_Id__c = fixtureId,
              score__c = '0-0',
              numberOfGoals__c = 0
            )
          );
        }
      }
    }
    //We insert the record
    insert matchToQuery;
  }
  //We give this method the actual match record, and the body from the response of the callout
  public static void otherCallouts(Match__c match, String jsonResponse) {
    //To work with it, I used to save the JSON inside a record, to avoid using too much of the API
    // Saved_Json__c s = new Saved_Json__c(JSON__c = jsonResponse, name = 'final');
    // insert s;

    //We transform the response body into a map
    Map<String, Object> responseMap = (Map<String, Object>) JSON.deserializeUntyped(
      jsonResponse
    );
    //There is one match, so one response here
    List<Object> responseList = (List<Object>) (responseMap.get('response'));

    //This was the previous version. We got the "away goals" and "home goals" directly from the API
    //And we used to put them inside a text field, but it didn't work during the final
    //Map<String, Object> firstRes = (Map<String, Object>) responseList[0];
    // Map<String, Object> goals = (Map<String, Object>) firstRes.get('goals');
    // String currentResult =
    //   String.valueOf(goals.get('home')) +
    //   '-' +
    //   String.valueOf(goals.get('away'));

    //Instead now, we count the total number of goals during the match
    //String scorerForNotification = '';
    Integer numberOfGoals = 0;

    //We don't have a node called 'events' now. We just have a normal list
    // List<Object> events = (List<Object>) firstRes.get('events');

    //For each event, we verify if a goal has been scored. If yes, we get the scorer's name, the time, and the team.
    //And also, count the total number of goals by incrementing the variable number of goals.
    for (Object eachEvent : responseList) {
      Map<String, Object> mappedEvent = (Map<String, Object>) eachEvent;
      if (mappedEvent.get('type') == 'Goal') {
        Map<String, Object> scorer = (Map<String, Object>) mappedEvent.get(
          'player'
        );
        Map<String, Object> scorerTime = (Map<String, Object>) mappedEvent.get(
          'time'
        );
        Map<String, Object> scorerTeam = (Map<String, Object>) mappedEvent.get(
          'team'
        );
        String scorerNameValue = String.valueOf(scorer.get('name'));
        String scorerTimeValue = String.valueOf(scorerTime.get('elapsed'));
        String scorerTeamValue = String.valueOf(scorerTeam.get('name'));
        //We increment the scorerForNotification variable, in order to(maybe) display it on the screen
        scorerForNotification +=
          scorerNameValue +
          ', ' +
          scorerTimeValue +
          ' for  ' +
          scorerTeamValue +
          '...';
        numberOfGoals += 1;
      }
    }
    //if the number of goals variable equals the field value, we stop the algorithm
    if (numberOfGoals == match.numberOfGoals__c) {
      return;
    }
    //if there is a change(ie: previously it was 0-0 on the record, and now it's 1-0), we update the record, and send the notification
    update new Match__c(Id = match.Id, numberOfGoals__c = numberOfGoals);
    sendGoalNotification(scorerForNotification);
  }

  //Here I've reused the method I had used for the jokes-getter development 
  public static void sendGoalNotification(String GoalEvent) {
    // We get the notification type Id
    CustomNotificationType notificationType = [
      SELECT Id, DeveloperName
      FROM CustomNotificationType
      WHERE DeveloperName = 'Football_Notification'
    ];

    // We create a new custom notification
    Messaging.CustomNotification notification = new Messaging.CustomNotification();

    //We add the announcement
    notification.setTitle('GOAL GOAL GOAL GOAAAAAAAAAAL');
    notification.setBody(GoalEvent);

    // We set the notification type Id with the value we got from the previous SOQL
    notification.setNotificationTypeId(notificationType.Id);

    //We've defined the redirection link to a page that doesn't exist. But it could be anything
    notification.setTargetId('000000000000000');

    // When everything is set, we send the notification. If we cannot, we catch the error and display it on the debug logs
    try {
      notification.send(new Set<String>{ UserInfo.getUserId() });
    } catch (Exception e) {
      System.debug('Problem sending notification: ' + e.getMessage());
    }
  }
}
{% endhighlight  %}
<p>When the code has been written, I launched it from the Anonymous developer console. It's not mandatory, I could have launched it from Setup->Apex classes->Schedule Apex. I still have a preference for the first way, because it's more precise, but I have to admit that the second one is more user-friendly. On both ways, I can ask Salesforce to launch my code exactly at 9PM(which was the time when the match had to begin).</p>
{% highlight java %}
System.schedule('First Job', '0 30 16 11 06 ?', new footballAPIHandler());
{% endhighlight %}
<br>
<p>And now...it should be working...</p>
<p>Finally, I finished to receive the Rodri's goal notification one hour after the end of the match.</p>
<br>
![Result](/Images/Football_API_Result.jpg)
<br>
<h3>Sources</h3>
<ul>
  <li><a href="https://github.com/selimhamidou/Football_API">Github repo</a></li>
</ul>

Sélim HAMIDOU


