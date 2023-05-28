---
layout: post
title: "I developed Salesforce With a Google API to display on a map the best pizzerias around me"
date: 2023-05-27 08:49:26 +0300
categories: jekyll update
---

<p>Hey! We have to agree. Pizza is one of the most beautiful and comforting food of the world. Everytime is an occasion to discover some new pizzerias. And to be honest, even if I am an adventurer(hmmm I am not so sure, let me check on my Trailhead profile...No, I am a Ranger now), I have to challenge myself even more, by finding pizzerias I wouldn't go to normally. Thus, I've developed this Lightning Web Component to allow me to find the best pizzerias around me. And, who knows everything about everything(apart from ChatGPT, I mean). Yes, Google! So, for this development, we will integrate Salesforce with an API from Google called Places API. I don't want to do some advertising their API, but I've found it very easy to use, with a very well-written documentation. Now that everything was said, let's move to the integration!</p>

<h3>Step one: Integration</h3>
<p>First, we have to sign up to the <a href="https://console.cloud.google.com/welcome/">Google for developers</a>. By doing so, you get some free credit, which will be more than enough to realize our tests. You can then go to this <a href="https://console.cloud.google.com/apis/credentials">link</a>to get your credentials.</p>
<p>As you may know, we will save these credentials on our Salesforce org, by using a custom metadata type. You will surely notice that, for the previous integrations, I've used a new custom metadata type object everytime. I shouldn't, it's time consuming, and it's better to have our data in the same place. So, instead of doing this now, we will create a new custom metadata type, and everytime we will have a new integration to do, we will only add a new record. </p>
![Twilio Console](/Images/Twilio_Credentials.jpg)
<p>When it's done, we go to the Remote site settings, and we authorize the endpoint we are going to use. Here the authorization is done from the endpoint, so the most straightforward way to do it is through a remote site setting. If we had to use an OAuth authentication for example, we would have used a named credential for this.</p>
![Twilio Custom Metadata Type Settings](/Images/Twilio_Credentials_Mdt_Settings.jpg)
<p>And now, let's make the magic happen!</p>

<h3>Step 2: The Apex part</h3>
<p>Now that the UI is set, what do we want? We want that, when we create an event, we send a SMS.
So, using a trigger is the thing to do. A flow could be used too, but in every cases, we will have to use some Apex code(it's especially true when we have to call an API), so we prefer to do everything in Apex instead of using a flow which will call some Apex code, but it's up to you to choose between both.
We also have to be aware of the fact that here, handling the events inserts is not sufficient. What do we do when we change the date of an event. Does the SMS have to be sent on the previous time? And when we delete the event?
For this, we check the context(before, after, update, insert, delete) on the trigger, and the we call the trigger handler.
Using a trigger handler is really good to keep our code clean.</p>

{% highlight java %}
//I know, we have to be careful with "after triggers", to avoid recursion. 
//Here, I used an after insert to be able to get the actual event Id, and to reinject it in the scheduled job's name. 
//By doing this way, I get unique job names
//The trigger works on insert(ie we create a new event), on update(ie we change the date and/or the time of the sms alert), and on delete(ie we delete the event, so we also have to delete the job. Anyway, it will still run, and we don't want this)
trigger onEventTrigger on Event(after insert, before update, before delete) {
  //We check the context: first to control exactly when the code triggers, and also to avoid recursion
  if (Trigger.isInsert && Trigger.isAfter) {
    eventTriggerHandler.onAfterInsert(Trigger.new);
  }
  //Now that the first import has already been done, it's simpler. The context here just needs a classical before update
  if (Trigger.isUpdate && Trigger.isBefore) {
    eventTriggerHandler.onBeforeUpdate(Trigger.New, Trigger.oldMap);
  }
  //When the user deletes an event, we have to avoid the job to be processed. So, we remove it.
  if (Trigger.isDelete) {
    eventTriggerHandler.onBeforeDelete(Trigger.New);
  }
}
{% endhighlight %}

<h4>The trigger handler</h4>
<p>As you guessed, all the methods you've seen called on the trigger are defined right here, on the handler.
For this handler, the biggest challenge was to remove the existing jobs when it was needed. I am sure you noticed that I didn't add some State filters on the SOQL I made for the cronTriggers. It's because if I did, I would have a bug everytime I would try to change the date of an event, when the SMS has already been sent(the State of the job is "deleted", but it's still on the database, so the new job can't have the same name). This part has still to be improved.</p>
{% highlight java %}
public with sharing class eventTriggerHandler {
  //This method will be called when it's needed(ie on Update and Delete contexts)
  public static Map<String, CronTrigger> getAllJobs() {
    List<CronTrigger> cronTriggers = [
      SELECT CronJobDetail.Name, State
      FROM CronTrigger WHERE CronJobDetail.Name LIKE '00U%'
    ];
    //Here we connect every event Id to the associated job. In this way, it's easier to retrieve them
    Map<String, CronTrigger> eventIdCronTriggerMap = new Map<String, CronTrigger>();
    for (CronTrigger eachCronTrigger : cronTriggers) {
      eventIdCronTriggerMap.put(
        eachCronTrigger.CronJobDetail.Name,
        eachCronTrigger
      );
    }
    return eventIdCronTriggerMap;
  }
  public static void onAfterInsert(List<Event> triggeredEvents) {
    //Even if the calendar is generally handled manually, some events could be sent by imports. 
    //So it's still important to bulkify our code
    for (Event e : triggeredEvents) {
      scheduleJob(e);   //This method is reusable, because the algorithm is the same on insert and update contexts
    }
  }
  public static void onBeforeUpdate(
    List<Event> triggeredEvents,
    Map<Id, Event> oldTriggeredEventsMap
  ) {
    Map<String, CronTrigger> eventIdCronTriggerMap = eventTriggerHandler.getAllJobs();
    for (Event e : triggeredEvents) {
      //We check if the user has changed the start date and/or the Send_a_SMS__c field
      if (
        (oldTriggeredEventsMap.get(e.Id).Send_a_SMS__c != e.Send_a_SMS__c) ||
        (oldTriggeredEventsMap.get(e.Id).StartDateTime != e.StartDateTime)
      ) {
        //We can't update a scheduled job, we can just delete it. So we check on the previous map if the job exists.
        //If yes, we have its Id, so we can remove the job
        if (eventIdCronTriggerMap.get(e.Id) != null) {
          System.abortJob(eventIdCronTriggerMap.get(e.Id).Id);
        }
        //Now we can create a new job, by calling the scheduleJob with the actual event as an entry
        scheduleJob(e);
      }
    }
  }

  //This method gets the number of hours on the Send_a_SMS__c field
  //For example, if the value of the field is "1 hour before", the method will return 1
  public static Integer extractNumberHours(String inputString) {
    Pattern regex = Pattern.compile('\\d+'); //We define our regex, which has to match one or more digits in a string
    Matcher matcher = regex.matcher(inputString); //We apply the regex to our string

    if (matcher.find()) {
      //If the match succeeds...
      String numberString = matcher.group(); //We get the found value
      return Integer.valueOf(numberString); //We convert it to an integer, and we return it to the scheduleJob method
    }
    return null; // We have found no number in the string
  }
  public static void scheduleJob(Event e) {
    //if e.Send_a_SMS__c is null, that means that the user doesn't need to receive a sms.
    //At first, I wanted to use a checkbox(send vs don't send), AND a picklist(if "send", when to send the sms).
    //But to simplify the UI, I decided to only use a picklist
    if (e.Send_a_SMS__c != null) {
      //We extract the number of hours before the event to send the sms
      //Then, we determine the actual datetime of when to send it
      Integer hoursBeforeMeeting = extractNumberHours(e.Send_a_SMS__c);
      Datetime SMSDateTime = e.StartDateTime.addHours(-hoursBeforeMeeting);
      //The scheduled job can't be in the past, so we add this condition to avoid some errors about the jobs
      if (SMSDateTime > Datetime.now()) {
        //These variables are saved to be dynamically reused on the string.format() method
        String minutes = String.valueOf(SMSDateTime.minute());
        String hour = String.valueOf(SMSDateTime.hour());
        String dayOfMonth = String.valueOf(SMSDateTime.day());
        String month = String.valueOf(SMSDateTime.month());
        String year = String.valueOf(SMSDateTime.year());
        String sch = string.format(
          '0 {0} {1} {2} {3} ? {4}',
          new List<string>{ minutes, hour, dayOfMonth, month, year }
        );
        //We schedule a new job, with the event Id as a Name
        twilioHandler m = new twilioHandler();
        String jobID = System.schedule(e.Id, sch, m);
      }
    }
  }
  public static void onBeforeDelete(List<Event> triggeredEvents) {
    Map<String, CronTrigger> eventIdCronTriggerMap = eventTriggerHandler.getAllJobs();
    for (Event e : triggeredEvents) {
      System.abortJob(eventIdCronTriggerMap.get(e.Id).Id);
    }
  }
}
{% endhighlight %}

<h4>The REST callout</h4>
<p>So, here we schedule a callout. For this, we need both the Schedulable interface, and a future method(callouts cannot be made from a Schedulable alone). By knowing this, we associate the current user phone, the credentials, and the Twilio phone, to make our callout. When it's done, we log a debug('success' or 'failure').</p>

{% highlight java %}
global with sharing class twilioHandler implements Schedulable {
  //Because we call an external service, we need to use a future(with the callout=true annotation), or a queueable
  @future(callout=true)
  global static void callAPI() {
    //Here the current user(ie the user doing an operation on the event) will receive the message.
    //But you can change it as you like.

    //We use the UserInfo.getUserId() to get its id, and we get its phone number with a SOQL query
    User currentUser = [
      SELECT Phone, Id
      FROM User
      WHERE Id = :UserInfo.getUserId()
    ];
    //We define an http request
    Http http = new Http();
    HttpRequest request = new HttpRequest();

    //We call the custom metadata type we created sooner(Twilio_Credentials__mdt), to get the actual token, the AccountSID 
    //and the sender phone number.
    //These values were given by the Twilio API, we use them in order to connect to the API
    String key = String.valueOf(
      Twilio_Credentials__mdt.getInstance('Send_SMS').get('Token__c')
    );
    String AccountSID = String.valueOf(
      Twilio_Credentials__mdt.getInstance('Send_SMS').get('Account_SID__c')
    );
    String phone = String.valueOf(
      Twilio_Credentials__mdt.getInstance('Send_SMS').get('Phone_Number__c')
    );
    //We define the header
    Blob credentials = Blob.valueOf(accountSID + ':' + key);
    String VERSION = '3.2.0';
    request.setHeader('X-Twilio-Client', 'salesforce-' + VERSION);
    request.setHeader('User-Agent', 'twilio-salesforce/' + VERSION);
    request.setHeader('Accept', 'application/json');
    request.setHeader('Accept-Charset', 'utf-8');
    request.setHeader(
      'Authorization',
      'Basic ' + EncodingUtil.base64Encode(credentials)
    );
    request.setHeader('Content-Type', 'application/x-www-form-urlencoded');

    //We define the endpoint
    request.setEndpoint(
      'https://api.twilio.com/2010-04-01/Accounts/' +
      AccountSID +
      '/Messages.json'
    );
    //We define the body
    request.setBody(
      'To=' +
      EncodingUtil.urlEncode(String.valueOf(currentUser.Phone), 'UTF-8') +
      '&From=' +
      EncodingUtil.urlEncode(phone, 'UTF-8') +
      '&Body=' +
      'An event is about to start. Please log in to your Salesforce org.'
    );
    //We need to post these data to the Twilio API, which will send the SMS to the user's phone, so the POST method is the one to use here
    request.setMethod('POST');

    //We send the request
    HttpResponse response = http.send(request);

    // If the request is successful, we parse the JSON response
    if (response.getStatusCode() == 201) {
      system.debug('success');
    } else {
      system.debug('failure');
    }
  }
  //We use the execute method of the Schedulable interface to call the Twilio API
  global void execute(SchedulableContext SC) {
    callAPI();
  }
}
{% endhighlight %}

<h3>Conclusion</h3>
<p>Now that everything is in place, we can receive a SMS everytime we add an event on the calendar, which is great! At this time, the message is not personalized, but it could be, by adding some other lines of code!</p>
![Twilio Result](/Images/Twilio_Result.jpg)

<h3>Sources</h3>
<ul>
  <li><a href="https://www.twilio.com/docs/usage/api?code-sample=code-send-a-simple-sms-using-the-programmable-sms-api&code-language=curl&code-sdk-version=json">Twilio API doc</a></li>
</ul>




