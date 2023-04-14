---
layout: post
title:  "How I integrated Salesforce with ChatGPT"
date:   2023-03-30 19:49:26 +0200
categories: jekyll update
---
Hey! The truth behind this article was that I was speaking with my team of developers, and we love trying new restaurants,
especially kebabs! But we were pretty unsure about which restaurant to give a try(we were in Paris). 
So I told myself it could be cool to integrate chatGPT with one of my sandboxes...

HTML part
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
          <!-- Default/basic -->
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

JavaScript Part:
Basically, in this part, we get the question asked from the html part, and it's saved in the variable questionToAsk.
This variable will be used to call the callChatGPT apex method.
The role of the javascript method `handleEnter` is simply to verify if the user typed the touch Enter. If he did, we call Apex. If he didn't, we don't.

{% highlight javascript %}
import { LightningElement, track } from "lwc";
import callChatGPT from "@salesforce/apex/BackEndChatGPT.callChatGPT";

export default class ChatGPTConsole extends LightningElement {
  @track data;
  @track error;
  questionToAsk;

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

  handleEnter(event) {
    if (event.key == "Enter") {
      this.questionToAsk = event.target.value;
      this.handleCallout();
    }
  }
}
{% endhighlight %}

Backend part

{% highlight java %}
public class BackEndChatGPT {
  public BackEndChatGPT() {
  }
  private static String CHAT_GPT_KEY = OpenAiKey__c.getValues('keyValue')
    .Key__c;
  private static final String ENDPOINT = 'https://api.openai.com/v1/chat/completions';

  @AuraEnabled
  public static String callChatGPT(String questionToAsk) {
    Http h = new Http();
    HttpRequest req = new HttpRequest();
    req.setEndpoint(ENDPOINT);
    req.setMethod('POST');
    String body =
      '{"model": "gpt-3.5-turbo","messages": [{"role": "user", "content": "' +
      questionToAsk +
      '"}], "temperature":1.2}';
    req.setHeader(
      'Authorization',
      'Bearer ' + CHAT_GPT_KEY
    );
    req.setHeader('content-type', 'application/json');
    req.setBody(body);
    HttpResponse res = h.send(req);
    if (res.getStatusCode() == 200) {
      Map<String, Object> results = (Map<String, Object>) JSON.deserializeUntyped(
        res.getBody()
      );
      List<Object> choices = (List<Object>) results.get('choices');
          Map<String, Object> dataNode = (Map<String, Object>)((Map<String, Object>)choices[0]);
          String content = String.valueOf(((Map<String, Object>)(dataNode.get('message'))).get('content'));
      return content;
    }
      return '';
  }
}
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
