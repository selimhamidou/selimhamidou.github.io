---
layout: post
title:  "How I integrated Salesforce with ChatGPT"
date:   2023-03-30 19:49:26 +0200
categories: jekyll update
---
Hey! The truth behind this article is that I was speaking with my team of developers, and we love trying new restaurants,
especially kebabs! But we were pretty unsure about which restaurant to give a try, every discussion about that was creating conflicts. So, I integrated ChatGPT with one of my Salesforce sandboxes to make this life changing choice... 

<h3>Configuration Part</h3>
First, we have to add a custom metadata type to store our variables. By doing this, Salesforce will store our sensitive data, and we will just have to access it from our code!

We create a custom metadata type called Credentials
![Creating Credentials Custom Metadata Type Step 1](/Images/credentials_mdt_step1.jpg)

We create two new fields of type text: one for the token, and the second for the endpoint
![Creating Credentials Custom Metadata Type Step 2](/Images/credentials_mdt_step2.jpg)

![Creating Credentials Custom Metadata Type Field Token ](/Images/credentials_mdt_create_token_field.jpg)

![Creating Credentials Custom Metadata Type Field Endpoint ](/Images/credentials_mdt_create_endpoint_field.jpg)

Here is the final state of our custom metadata type
![Creating Credentials Custom Metadata Type Final](/Images/credentials_mdt_final.jpg)

To get a token, you have to go to this link from the API we are using, and ask for a token:`https://platform.openai.com/account/api-keys`
Then, copy the token in your Notepad.
![Getting a token from OpenAI](/Images/credentials_mdt_final.jpg)

Now we add a record on our custom metadata type, with `https://api.openai.com/v1/chat/completions` as an endpoint, and with the token we just got. 
![Creating Credentials Custom Metadata Type Record For ChatGPT](/Images/credentials_mdt_chatgpt_record.jpg)

We also have to add a remote site setting, to tell Salesforce that our endoint is legit, so we go to Setup -> Remote Site Settings -> New:
![Creating a remote site setting for the endpoint](/Images/create_remote_site_setting.jpg)

Now that everything is configured properly, we can move to the development part.

<h3>HTML Part</h3>
Ths HTML part is pretty simple: we are drawing a card, and inside of it, we are adding an input("which are the best kebabs in Paris?) and an output("the best kebabs are x, y and z", which will be given by chatGPT itself!).

{% highlight html %}
<template>
  <article class="slds-card">
    <div class="slds-card__header slds-grid"></div>
    <div class="slds-card__body slds-card__body_inner">
      <h2 class="slds-text-heading_medium slds-m-bottom_medium">
        Ask anything to ChatGPT.
      </h2>
      <div class="slds-form-element">
        <div class="slds-form-element__control slds-border_bottom">
          <div class="slds-form-element__static">
            <p>{data}</p>
          </div>
          <div class="slds-p-around_medium lgc-bg">
            <lightning-input
              value={questionToAsk}
              onenter={handleCallout}
              onkeypress={handleEnter}
              type="text"
              label="Enter some text"
            ></lightning-input>
          </div>
        </div>
      </div>
    </div>
  </article>
</template>
{% endhighlight %}

<h3>JavaScript Part</h3>
Basically, in this part, we get the question asked from the html part, and it's saved in the variable questionToAsk.
This variable will be used to call the callChatGPT apex method.
The role of the javascript method `handleEnter` is simply to verify if the user typed the touch Enter. If he did, we call Apex. If he didn't, we don't.

{% highlight javascript %}
import { LightningElement, track } from "lwc";
import callChatGPT from "@salesforce/apex/BackEndChatGPT.callChatGPT";

export default class ChatGPTConsole extends LightningElement {
  //These two variables will be used to handle the response from ChatGPT
  @track data;
  @track error;

  questionToAsk; //Gotten from the lightning-input

  //We don't call ChatGPT until the user presses enter
  handleEnter(event) {
    if (event.key === "Enter") {
      this.questionToAsk = event.target.value;
      this.handleCallout();
    }
  }
  //When the user did, we call our Apex method(with questionToAsk as a parameter), and we display the response(data) in the html
  handleCallout() {
    callChatGPT({ questionToAsk: this.questionToAsk })
      .then((result) => {
        this.data = result;
        console.log("returned data: " + data);
      })
      .catch((error) => {
        this.error = error;
        console.log("there is an error: " + error);
      });
  }
}
{% endhighlight %}

<h3>Apex Part</h3>
{% highlight apex %}
public class BackEndChatGPT {
  
  //We get the token and the endpoint from the custom metadata type Credentials__mdt, that we created before
  //Using custom metadata type is a personal choice. I could have used custom setting instead, it would have worked the same
  private static String token = Credentials__mdt.getInstance('ChatGPT')
    .Token__c;
  private static String endpoint = Credentials__mdt.getInstance('ChatGPT')
    .Endpoint__c;

  //This method is the one called by our LWC
  //We call this method imperatively, so no need to add cacheable=true, but it's better for the performance
  @AuraEnabled(cacheable=true)
  public static String callChatGPT(String questionToAsk) {
    //We create a POST request, and we define some parameters to give to the ChatGPT API. The model will be defined by our usage.
    Http h = new Http();
    HttpRequest req = new HttpRequest();
    req.setEndpoint(endpoint);
    req.setMethod('POST');
    String body =
      '{"model": "gpt-3.5-turbo","messages": [{"role": "user", "content": "' + //The model gpt-3.5-turbo seems to be the most versatile to use
      questionToAsk +
      '"}], "temperature":0.7}'; //The temperature defines the precision of the response. It's an arbitrary value, we could use more, or less(but always between 0 and 2)
    req.setHeader('Authorization', 'Bearer ' + token);
    req.setHeader('content-type', 'application/json');
    req.setBody(body);
    HttpResponse res = h.send(req);
    //We check if the request has succeeded. If yes, we treat the response, and give it back to our LWC.
    //If no, we simply return an empty string
    if (res.getStatusCode() == 200) {
      Map<String, Object> results = (Map<String, Object>) JSON.deserializeUntyped(
        res.getBody()
      );
      List<Object> choices = (List<Object>) results.get('choices');
      Map<String, Object> dataNode = (Map<String, Object>) ((Map<String, Object>) choices[0]);
      String content = String.valueOf(
        ((Map<String, Object>) (dataNode.get('message'))).get('content')
      );
      return content;
    }
    return '';
  }
}
{% endhighlight %}

