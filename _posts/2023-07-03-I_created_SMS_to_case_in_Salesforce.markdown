---
layout: post
title: "I developed SMS-To-Case in Salesforce"
date: 2023-07-04 09:00:00 +0300
categories: jekyll update
---


Hey! I am proud of this one. I am betting that in every project you have been working on, you had to work with email to case. 

I know that Email-to-case is a popular feature right now, but...what about SMS-To-Case? We all have a mobile phone in our pocket. It would be great if we could reach the customer service by SMS when a product we bought doesn’t work as expected. After the interaction, a case would be created, or at least handled, as it is already the case with Email-To-Case. Unfortunately, this feature doesn’t exist yet, so, let’s remediate this.


### Step one: Let's think for a minute...How we can do it?

You agree that, if we had to develop a new Email-to-case, we would use an inbound email handler(which is just an Apex class that is doing something every time your Salesforce org is getting an Email(to an address we would have defined before)).</p>

Ok, now, what is the difference between Email-to-case and SMS-to-case? It has no differences. You receive a message from a user, and you handle it. Ok, I am oversimplifying it, but that's the truth. We get a message from somewhere, and we use it to create a case. Now, what is this "somewhere"? 

To answer this last question, we have to know what a webhook is. A webhook is a function that will send a notification to a URL every time an event is firing. It's like platform events, but not in Salesforce. Webhooks are not something we can use with every API. For example, it's not available with the <a href="https://www.selimhamidou.com/posts/I_tried_to_receive_on_Salesforce_real_time_UCL_final_notifications_and_I_failed">football API I was using to receive goal notifications during the last UCL Final.</a>

To determine which API to use to create this solution, we have to wonder which API can both send and receive SMS. Twilio is the simplest solution to do it. That means that we can send an SMS to a specific phone number, and it will be received by Twilio. And the last thing, webhooks are supported. So, Twilio looks like the perfect solution for our needs.


<h3>Wow! How do we begin?</h3>

To integrate Salesforce with Twilio API, and if you have never developed a solution to <a href="https://www.selimhamidou.com/posts/I_Developed_A_Solution_To_Receive_SMS_Alert_Before_A_Meeting">handle SMS alerts on Salesforce</a>, I invite you to follow <a href="https://www.selimhamidou.com/posts/I_Developed_A_Solution_To_Receive_SMS_Alert_Before_A_Meeting">this link</a>. It will show you step by step how to connect your Salesforce organization with Twilio API.

When it's done, you can move to the <a href="https://console.twilio.com/us1/develop/phone-numbers/manage/incoming/">Twilio console</a>. There, you can manage your Twilio phone numbers. You can also create new numbers by buying some. I didn't do it, but as a company, and for pricing reasons, maybe you would prefer to use a phone number from your own country. It's doable with Twilio API. 

So you click on the phone number you want to use for this development, and you go to the "Messaging Configuration" section. There you can define your webhook URL. To know the URL we are using as a webhook endpoint, we have to go back to Salesforce for this.


![Twilio settings](/Images/SMS_to_case_twilio_setting.jpg)


![Twilio configuration](/Images/SMS_to_case_twilio_config.jpg)


Let's recap a little bit: a user is sending an SMS to a Twilio number. Boum, there is an event on Twilio. Now, we got to send this event to an URL, but what URL? It's really simple. We create a public site. This site will have an URL and will be associated with a web service. That means that every time we "visit" this website(not a visit, I mean every time we send a request to this website), some Apex code is executed. So, let's go! Let's create this site!


![Public site creation](/Images/SMS_to_case_site.jpg)


![Public site endpoint](/Images/SMS_to_case_site_domain.jpg)


The template doesn't have any importance right here. The role of the website is just to provide an address we can use. You can copy and paste this address to the Twilio console. You also have to add '/services/apexrest/' at the end of the endpoint.


### Handling Twilio webhook verification
This part is really important. Imagine that you are someone mean and that you want to post some wrong data on my Salesforce organization. We want to avoid this. But how? With the Apex Crypto class. I took some time to understand this concept, but in fact, it's pretty simple: Twilio is sending you some data. With its data, it's giving you a signature on the header, basically saying "Hey, that's me!". Our job now is to calculate the key from the elements you got and compare both. If both have the same signature, that means that the entity which is trying to connect with us is Twilio, and not someone else. But still, be careful: the encryption algorithm and data involved are not the same with every API. You have to read the documentation first.


<img src="https://assets.cdn.prod.twilio.com/images/sms-http-request-cycle.width-800.gif">


Here is the Apex code. We send an SMS to Twilio. An event is created on Twilio. It fires Twilio's webhook, which is sending a POST request to our web service. That's good news because our web service has precisely a method to handle POST callouts. So the first thing we do is verify if the request is legit. If it's not, we simply throw an error. If it is, we look for a case number on the SMS body. If a current case number exists on the SMS body, we reopen the case and add some information to this case. If it doesn't exist, we create a new case with the information we get from the SMS. In all cases, we send a response by SMS(the SMS model is stored inside a custom label).


![SMS template](/Images/SMS_to_case_customLabel.jpg)


{% highlight apex %}
//We define a web service. Every page after my current endpoint will be accepted(*)
@RestResource(urlMapping='/*')
global class webHookHandler {
  //We want to do something every time Twilio API is posting something to our web service
  @HttpPost
  global static void handleNotifications() {
    //We use a try-catch to handle all the exceptions we could get
    try {
      //First step: we verify is the entity that is trying to connect to our API is actually Twilio(VERY VERY IMPORTANT)
      //We got a request from somewhere
      RestRequest request = RestContext.request;
      RestResponse response = RestContext.response;
      //We get its signature. Its name is X-Twilio-Signature here
      String hashedVal = request.headers.get('X-Twilio-Signature');
      //The request parameters will be used to recalculate the actual signature
      Map<String, String> params = request.params;
      //We call a function to recalculate the signature
      String calculatedSignature = computeSignature(
        'https://selimhamidou-dev-ed.my.salesforce-sites.com/services/apexrest/',
        params
      );
      //If the recalculated signature and the one which is given by Twilio are the same, we can make the SMS to casework. If not, we don't
      if (calculatedSignature == hashedVal) {
        String fromValue = request.params.get('From');
        String bodyValue = request.params.get('Body');
        //Here we are looking for the case number inside the SMS's body. If we find it and it's correct(ie if a case exists with this number), we update the case
        //If not, we create a new one
        String foundCaseNumber = findCaseNumberInsideBody(bodyValue);
        if (foundCaseNumber != null) {
          List<Case> cList = [
            SELECT Id, CaseNumber, Description
            FROM Case
            WHERE CaseNumber = :foundCaseNumber
          ];
          if (cList.size() > 0) {
            updateCase(cList, bodyValue, fromValue);
          } else {
            createCase(bodyValue, fromValue);
          }
        } else {
          createCase(bodyValue, fromValue);
        }
      } else {
        //If the entity trying to log in to our web server is not Twilio, it receives an error
        response.statusCode = 401;
        CalloutException e = new CalloutException();
        e.setMessage('Unauthorized access.');
        throw e;
      }
    } catch (Exception e) {
      system.debug('Error: ' + e.getMessage());
    }
  }
  //With this method, we research with a Regex the number of a request inside the SMS body
  public static String findCaseNumberInsideBody(String body) {
    Pattern pattern = Pattern.compile('\\d{8}');
    Matcher matcher = pattern.matcher(body);

    if (matcher.find()) {
      String matchedString = matcher.group();
      return matchedString;
    } else {
      return null;
    }
  }

  //With this method, we recalculate Twilio's signature.
  //The URI is the exact URL we gave to Twilio
  //The params map could change over time. To avoid any problem in the future, we are getting this map from Twilio's request header
  public static String computeSignature(
    String uri,
    Map<String, String> params
  ) {
    //We verify if we get some parameters on the request's header
    if (params != null) {
      //We sort all the parameter names in an alphabetical number
      List<String> paramNames = new List<String>(params.keySet());
      paramNames.sort();

      //We do the same with the values
      for (String paramName : paramNames) {
        List<String> values = getValues(params, paramName);
        values.sort();

        //We concatenate the parameter names and values
        for (String value : values) {
          uri += paramName + value;
        }
      }
    }
    //We also need the auth token, which is used as a secure key by Twilio API
    String token = String.valueOf(
      Twilio_Credentials__mdt.getInstance('Send_SMS').get('Token__c')
    );

    //We calculate the key
    Blob mac = Crypto.generateMac(
      'HmacSHA1',
      Blob.valueOf(uri),
      Blob.valueOf(token)
    );
    //We encode it in base 64
    String computed = EncodingUtil.base64Encode(mac);

    //We remove any space that we can get
    return computed.trim();
  }

  private static List<String> getValues(
    Map<String, String> paramDict,
    String paramName
  ) {
    if (paramDict.containsKey(paramName)) {
      return new List<String>{ paramDict.get(paramName) };
    } else {
      return new List<String>();
    }
  }
  //We use this method to create a case, by getting the SMS text, and the phone number
  public static void createCase(String bodyValue, String fromValue) {
    Case createdCase = new Case(Status = 'Open', Description = bodyValue);
    insert createdCase;
    String caseNumber = String.valueOf(
      [SELECT Id, CaseNumber FROM Case WHERE Id = :createdCase.Id]
      .CaseNumber
    );
    String message = String.format(
      System.label.SMS_to_case_message,
      new List<String>{ caseNumber }
    );
    twilioHandler.callAPI(message, fromValue);
  }
  //We use this method to update a case, by getting the SMS text, and the phone number
  public static void updateCase(
    List<Case> cList,
    String bodyValue,
    String fromValue
  ) {
    Case c = cList[0];
    c.Status = 'Open';
    c.Description += ' - ' + bodyValue;
    update c;
    String message = String.format(
      System.label.SMS_to_case_message,
      new List<String>{ c.CaseNumber }
    );
    twilioHandler.callAPI(message, fromValue);
  }
}

{% endhighlight %}


<br>
<h3>Sources</h3>
<ul>
<li><a href="https://www.twilio.com/docs/usage/webhooks/getting-started-twilio-webhooks">Twilio webhooks documentation</a></li>
<li><a href="https://www.youtube.com/watch?v=weoh7i5UCI4">A video from Meera Nair, very instructive to understand how webhooks work with Salesforce.</a></li>
<li><a href="https://developer.salesforce.com/docs/atlas.en-us.apexref.meta/apexref/apex_classes_restful_crypto.htm">Documentation about encrypting data in Salesforce.</a></li>
</ul>


Sélim HAMIDOU

