---
layout: post
title: "I boosted the productivity of my service users with this simple apex code"
date: 2023-06-21 19:55:00 +0200
categories: jekyll update
---

<p>Hi! Let's talk about productivity today. We all have our proper idea about what is productivity. Elon Musk, for example(I don't know him, so don't reach me on Linkedin to get his phone number, I don't have it!), objectively, looks productive. According to the internet(which could not be accurate), he may do weeks of 80-100 hours, which is huge. Concerning myself, I am not always as productive as I want. Sometimes, I write two articles in a week, and some other times, zero. I am still convinced that it's not a bad thing to be not productive every time. I also like the idea that we can build anything, whether it's useful or not. For example, I am pretty much convinced that this article won't have any impact on our lives. But I enjoyed writing it! By reading this, please, don't take it literally. Even if work is taking a huge part in our lives, it's just work. It can be fun sometimes, or painful some other times, but we are not all doctors, aircraft pilots, or other life-saving job. I am proud of what I do and who I am, but let's be honest: I am just writing some lines on my computer, and say 'Tadaaaaaa' when it's working. And sometimes it isn't! </p>

<h3>Step one: What do we want</h3>
<p>We want to boost the productivity of the service users. So, to begin, we have to ask ourselves two questions: 
    <ul>
        <li>How to measure productivity?</li>
        <li>What to do when productivity is low?</li>
    </ul>
To measure productivity, I have been thinking about getting the average time a specific user has been scrolling on Salesforce. Some tools can make it possible, but installing a new package on our org would make our implementation a little less straightforward. The most straightforward solution for me is to calculate the number of closed cases and compare it with the total number of assigned cases. If the user has a low number of closed cases, 'we are not happy'. Now, what could we do when we are not happy with him? We send him an SMS. 4 PM is the perfect hour because the day is not finished, but at the same time, the day is almost finished. Yes, I know, it's not something we want to see anywhere. But it's still interesting to see how it's working, so be ready!</p>

<h3>The beginning of everything</h3>
<p>Do you know what is missing on the case object on Salesforce? A field telling us the date when a Case has been opened for the last time. I've searched about this field and didn't find it. The Date/Time Opened only works on the case's creation. So I created a field called Datetime_of_last_opening__c, of Datetime type. We can't use a formula field here(which would have been useful), because the functions ISCHANGED and PRIORVALUE don't exist on them for now. So instead, we have used an Apex trigger. If you want, you can do this step with a flow, it would work. And even more, it's considered a "best practice" by Salesforce to use declarative tools whenever you can. In my case, I preferred to use apex code from the beginning until the end, to be able to optimize the performance of the algorithm(maps are not available yet on flows), and to keep all the logic in the same place(which is also a "best practice")</p>
![Field](/Images/productivity_booster_field.jpg)

<p>Now that the field is created, we have to give it some data. We moved all the logic inside a handler that we called CaseManager. Doing this allows us to get a cleaner trigger. In the trigger, we only verify the context(before, insert, update...), and according to the actual context, we call the method we want. We also don't need to define a list and make a proper insert or update because we are modifying the current record during the transaction, which is (again) simplifying the logic.</p>

{% highlight java %}
trigger CaseTrigger on Case(before insert, before update) {
  //We check the context: I want the Datetime_of_last_opening__c to work on insert and update.
  //I also don't need the record Id, so a "before" context is more than enough
  if (Trigger.isUpdate) {
    CaseManager.onBeforeUpdate(Trigger.New, Trigger.oldMap);
  }

  if (Trigger.isInsert) {
    CaseManager.onBeforeInsert(Trigger.New);
  }
}
{% endhighlight %}
<p>Now, when the methods onBeforeUpdate and onBeforeInsert have done their job(which means 'saving into a field the last time a user(or a machine) opened the case'), we can move to the rest of the logic. The first thing to do is to get all the case records. We are interested in pretty much every case record, except for those that have been closed before today. And by using the returned list of cases, the ultimate goal is simply to get a map, with the agent's phone number as a key, and the percentage of closed cases as a value. Also, I've created a Profile called 'Customer Service User', but it's up to you to call it as you want.</p>
<p>To lower the amount of data involved, we only saved into the map the agents who don't perform(ie who didn't close 80% of their cases). Also, the percentage in the map is a String, not an Integer, to reuse it on the SMS message. And the last thing, we consider that the most straightforward way to develop our solution is to want to automate the Sending SMS task. For this, we will need to implement the Schedulable interface. We already integrated Salesforce with the Twilio API <a href="https://www.selimhamidou.com/posts/I_Developed_A_Solution_To_Receive_SMS_Alert_Before_A_Meeting">to send some alerts before meetings</a>. We simply improved this development, to reuse it. For doing it, we transformed some variables into the method's parameters.</p>
{% highlight java %}
public with sharing class CaseManager implements Schedulable {
  String userPhoneNumber;

  public static void onBeforeUpdate(
    List<Case> trgNew,
    Map<Id, Case> trgOldMap
  ) {
    //We don't have a lot of imagination for this. So, we call a variable "c", that we increment.
    //If you are more imaginative, it will be better to use understandable names
    for (Case c : trgNew) {
      //When the status moves from 'Non-closed' to 'Closed', we do what?...
      if (trgOldMap.get(c.Id).Status == 'Closed' && c.Status != 'Closed') {
        //We save the current date and time into a variable. It will be the last DateTime the Case has been (re)opened
        c.Datetime_of_last_opening__c = Datetime.now();
      }
    }
  }

  public static void onBeforeInsert(List<Case> trgNew) {
    //We also need this on insert. Maybe a case would be created with the status 'Closed', so we want to avoid this kind of situation
    //Almost the same logic here
    for (Case c : trgNew) {
      //Note: In my case, I only have one type of 'Closed' status, so I only used this one on my conditions.
      //You have to consider this if you choose to implement this development.
      if (c.Status != 'Closed') {
        c.Datetime_of_last_opening__c = Datetime.now();
      }
    }
  }

  //This method is maybe the most important one here: it has to get all the open cases that have been closed today
  //And those which have not been closed yet
  public static List<Case> getOpenedCases() {
    return [
      SELECT Id, Owner.Phone, Status, ClosedDate, Datetime_of_last_opening__c
      FROM Case
      WHERE
        (ClosedDate = today
        OR (Datetime_of_last_opening__c <= today
        AND Status != 'Closed'))
        AND Owner.Profile.Name = 'Customer Service User'
    ];
  }
  //We transform the list of cases to a map: the key will be the phone number of the Service user, and the value will be the percentage of closed cases today
  public static Map<String, String> getProductivityMap(
    List<Case> casesOfTheDay
  ) {
    Map<String, Integer> numberOfTreatedCases = new Map<String, Integer>(); //This map gives us the total number of treated cases
    Map<String, Integer> totalCasesByUser = new Map<String, Integer>(); //This one is for the total number of assigned cases

    //Basically, we count the number of treated cases and the total number of cases
    for (Case c : casesOfTheDay) {
      String userPhoneNumber = String.valueOf(c.Owner.Phone);
      if (c.Status == 'Closed') {
        if (numberOfTreatedCases.containsKey(userPhoneNumber)) {
          numberOfTreatedCases.put(
            userPhoneNumber,
            numberOfTreatedCases.get(userPhoneNumber) + 1
          );
        } else {
          numberOfTreatedCases.put(userPhoneNumber, 1);
        }
      }

      if (!totalCasesByUser.containsKey(userPhoneNumber)) {
        totalCasesByUser.put(userPhoneNumber, 0);
      }

      totalCasesByUser.put(
        userPhoneNumber,
        totalCasesByUser.get(userPhoneNumber) + 1
      );
    }

    Map<String, String> percentageOfClosedCases = new Map<String, String>();
    //When it's done, we calculate the percentage for every user
    for (String phoneNumber : numberOfTreatedCases.keySet()) {
      Integer closedCases = numberOfTreatedCases.get(phoneNumber);
      Integer totalCases = totalCasesByUser.get(phoneNumber);
      //We don't want to get a 33,333333...%, so we convert it to an integer
      Integer percentage = (Decimal.valueOf(closedCases) /
        decimal.valueOf(totalCases) *
        100)
        .intValue();
      //People with a percentage of more than 80% are considered "good workers". So we don't need to send them a message
      if (percentage <= 80) {
        percentageOfClosedCases.put(phoneNumber, String.valueOf(percentage));
      }
    }
    //We return the final map, and we will use it inside of the callTwilio method
    return percentageOfClosedCases;
  }
  //We use it to call the Twilio API with the Schedulable interface
  public static void execute(SchedulableContext SC) {
    //We define the messageToSend and productivityMap variables. I know, productivityMap is a collection instead
    String messageToSend;
    //We call the getProductivityMap to get all the numbers of workers we want to call
    Map<String, String> productivityMap = getProductivityMap(getOpenedCases());
    for (String eachNumber : productivityMap.keySet()) {
      //We define the message you will see on your mobile phone if you aren't productive enough
      messageToSend = string.format(
        'Wake up!!! Today you have closed only {0} % of your cases, which is very low...',
        new List<string>{ productivityMap.get(eachNumber) }
      );
      //We call the Twilio API
      twilioHandler.callAPI(messageToSend, eachNumber);
    }
  }
}
{% endhighlight %}

<p>Note: As I said, the Twilio integration pattern I used is the same as the one I used to integrate Twilio API with Salesforce in previous development. The only difference is that I've quickly rewritten my methods to be more easily reusable(basically by transforming some variables into method parameters. Let's make room for...THE NEW TWILIOHANDLER CLASS!!! Yes, I know, this class is so much better than the previous one.</p>

{% highlight java %}
global with sharing class twilioHandler implements Schedulable {
  //Because we call an external service, we need to use a future(with the callout=true annotation), or a queueable
  @future(callout=true)
  global static void callAPI(String message, String ToUser) {
    //We define an http request
    Http http = new Http();
    HttpRequest request = new HttpRequest();

    //We call the custom metadata type we created sooner(Twilio_Credentials__mdt), to get the actual token, the AccountSID
    //and the sender's phone number.
    //These values were given by the Twilio API, we use them to connect to the API
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
      EncodingUtil.urlEncode(ToUser, 'UTF-8') +
      '&From=' +
      EncodingUtil.urlEncode(phone, 'UTF-8') +
      '&Body=' +
      message
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
    //Here the current user(ie the user doing an operation on the event) will receive the message.
    //But you can change it as you like.
    //We use the UserInfo.getUserId() to get its id, and we get its phone number with a SOQL query
    User currentUser = [
      SELECT Phone, Id
      FROM User
      WHERE Id = :UserInfo.getUserId()
    ];
    String ToUser = String.valueOf(currentUser.Phone);
    callAPI(
      'An event is about to start. Please log in to your Salesforce org.',
      ToUser
    );
  }
}

{% endhighlight %}

<p>Ok, now that everything is in place, we just have to send an SMS at 4 PM every day, from Monday to Friday</p>
{% highlight java %}
System.schedule('Send SMS to service users', '0 27 15 ? * MON-FRI', new CaseManager());
{% endhighlight %}

<p>Now, let's see the result:</p>
![Now the result](/Images/productivity_booster_result.jpeg)
<br>
<h3>Sources</h3>
<ul>
  <li><a href="https://help.salesforce.com/s/articleView?id=sf.customize_functions_ischanged.htm&type=5">ISCHANGED on formula fields</a></li>
  <li><a href="https://help.salesforce.com/s/articleView?id=sf.customize_functions_priorvalue.htm&type=5">PRIORVALUE on formula fields</a></li>
  <li><a href="https://developer.salesforce.com/docs/atlas.en-us.apexcode.meta/apexcode/apex_scheduler.htm">How to schedule apex code</a></li>
</ul>

Sélim HAMIDOU