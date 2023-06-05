---
layout: post
title: "How to handle pagination on an LWC"
date: 2023-04-28 08:49:26 +0300
categories: jekyll update
---

<p>I was working on a project with a client, and they had a great idea: some of our users need to manipulate some records without having to go to the object itself. Having 100 or 1000 records on the same page would not be user-friendly. So, at this time, I had developed an Aura component that was fully configurable from the app builder. Now, I am doing it in LWC.</p>

<p>Before I begin, you have to know that you have a lot of possibilities with pagination (each solution has its positive points and drawbacks). I chose (at the time of the project, and right now too) to use OFFSET for that because I find it very straightforward, but I am aware of the limitations of this solution (especially concerning the multiple SOQL queries you do, which is impacting the performance of our LWC more than governor limits AND also the governor limits (you can only use OFFSET clause for ranges of 2000 records or less)). Knowing that, let's begin anyway...</p>

<h3>HTML Part</h3>
<p>We first build the skeleton of our LWC. Nothing really magical here, I've just used a model from Lightning Design System. The columns have to get some data and the column names. We will get these on our JavaScript file. From this model, I added four buttons to be able to change pages as we like.</p>

{% highlight html %}
<template>

  <article class="slds-card">
    <div class="slds-card__header slds-grid">
      <header class="slds-media slds-media_center slds-has-flexi-truncate">
        <div class="slds-media__figure">
          <span
            class="slds-icon_container slds-icon-standard-account"
            title="account"
          >
            <svg class="slds-icon slds-icon_small" aria-hidden="true">
              <use
                xlink:href="/assets/icons/standard-sprite/svg/symbols.svg#account"
              ></use>
            </svg>
            <span class="slds-assistive-text">{objectName}</span>
          </span>
        </div>
        <div class="slds-media__body">
          <h2 class="slds-card__header-title">
            <a
              href="#"
              class="slds-card__header-link slds-truncate"
              title={objectName}
            >
              <span>{objectName}</span>
            </a>
          </h2>
        </div>
      </header>
    </div>
    <div class="slds-card__body">
      <div style="height: 300px">
        <lightning-datatable key-field="id" data={data} columns={columns}>
        </lightning-datatable>
      </div>
      <div
        class="slds-grid slds-align_absolute-center"
        style="margin-top: 1rem"
      >
        <button
          class="slds-button slds-button_neutral"
          onclick={handleFirst}
          disabled={disablePrevButtons}
        >
          First
        </button>
        <button
          class="slds-button slds-button_neutral"
          onclick={handlePrevious}
          disabled={disablePrevButtons}
        >
          Previous
        </button>
        <button
          class="slds-button slds-button_neutral"
          onclick={handleNext}
          disabled={disableNextButtons}
        >
          Next
        </button>
        <button
          class="slds-button slds-button_neutral"
          onclick={handleLast}
          disabled={disableNextButtons}
        >
          Last
        </button>
      </div>
    </div>
  </article>
</template>
{% endhighlight %}

<h3>JavaScript file</h3>
<p>The first step here is getting our request(given by the user), and the number of results per page. Right now the request has to be simple, with one "SELECT", and one "FROM". Optionally, the "WHERE" will work too, but we can't(now, at least) use nested SOQL.</p>

{% highlight javascript %}
//We import all our apex methods and the keyword we are going to use
import { LightningElement, api, wire, track } from "lwc";
import getTotalNumberOfResults from "@salesforce/apex/queryClass.getTotalNumberOfResults";
import getResultsFromQuery from "@salesforce/apex/queryClass.getResultsFromQuery";
import getFieldsAndObjectNamesFromSoql from "@salesforce/apex/queryClass.getFieldsAndObjectNamesFromSoql";

export default class Datatable extends LightningElement {
//These two variables are directly set on the app builder, this is why I am using the api keyword
@api query;
@api ResultsPerPage;
//These two will be used to populate the data table
@track columns = [];
@track data = [];

//Variables used when the wire service doesn't get data
errorOnData;
errorOnFields;
errorOnNumberOfResults;

pageNumber = 1; //When the page is loaded, pageNumber = 1. After this, pageNumber can be changed(by clicking on "Next" for example), which will trigger our wire service
totalNumberOfResults;
objectName; //the name is retrieved from a regex in the apex method getFieldsAndObjectNamesFromSoql

//This getter is not necessary in this situation, because the totalNumberOfPages is not going to change.
//But it could be changing in the future if we add a feature to allow users to delete records for example
@api
get totalNumberOfPages() {
return Math.ceil(this.totalNumberOfResults / this.ResultsPerPage);
}
//We get the total number of results only to be able to handle the
@wire(getTotalNumberOfResults, { query: "$query" })
  wiredTotalNumberOfResults({ error, data }) {
    if (data) {
      this.totalNumberOfResults = data;
      this.errorOnNumberOfResults = undefined;
    } else if (error) {
      this.totalNumberOfResults = undefined;
      this.errorOnNumberOfResults = error;
    }
  }
//We get all the fields and object names of the request => if the query is //SELECT id, name from Account, it will return a list with Id, Name, Account inside of it
  @wire(getFieldsAndObjectNamesFromSoql, {
    query: "$query"
})
wiredFieldNames({ error, data }) {
if (data) {
const dataClone = [...data]; // We clone the list to be able to manipulate our data
this.objectName = dataClone.pop(); //We remove the last index(the object name) and we save them into a variable.
//We will only use it to display it as a title. The other indexes of the list will be used for the data table
this.columns = dataClone.map((fieldName) => {
//We create the columns we are gonna use for the data table
return {
label: fieldName,
fieldName: fieldName,
type: "text"
};
});
this.errorOnFields = undefined; //If data, we don't have any error
} else if (error) {
this.columns = undefined; //If error, we have no column, but an error that we are saving!
this.errorOnFields = error;
}
}

// With this method, we are doing a SOQL query every time the page changes. The $ sign allows us to handle the case when one of our variables is null
@wire(getResultsFromQuery, {
query: "$query",
    ResultsPerPage: "$ResultsPerPage",
pageNumber: "$pageNumber"
})
//Same logic here: if we get data, we save it. If not, we don't and save //the error
wiredData({ error, data }) {
if (data) {
this.data = data;
this.errorOnData = undefined;
} else if (error) {
this.errorOnData = error;
console.error(error);
}
}

handleNext() {
if (this.pageNumber < this.totalNumberOfPages) {
//We don't want to have an unlimited number of pages, the maximum number is calculated on the totalNumberOfPages variable
this.pageNumber += 1; //If we click on "Next", we move to the page after
}
}
handlePrevious() {
if (this.pageNumber > 1) {
//We don't want to have a negative number of pages, the minimum number is 1
this.pageNumber -= 1; //If we click on "Previous", we move to the page before
}
}
handleFirst() {
this.pageNumber = 1; //If we click on "First", we move to the first page
}
handleLast() {
this.pageNumber = this.totalNumberOfPages; //If we click on "Last", we move to the last page(calculated above)
}

@api
get disablePrevButtons() {
//Allows us to disable the "First" and "Previous" buttons when it's needed(ie when we are on page 1)
return this.pageNumber === 1;
}

// Method to disable the "Next" and "Last" buttons
@api
get disableNextButtons() {
//Allows us to disable the "Next" and "Last" buttons when it's needed(ie when we are on the last page for example)
return (
this.pageNumber === this.totalNumberOfPages ||
this.totalNumberOfPages === 0
);
}
}
{% endhighlight %}

<h3>Apex Part</h3>
<p>There are three methods here. The first method, getResultsFromQuery, allows us to retrieve the results. Every time the page is changed (by clicking on "Next", for example), this method is called with a different OFFSET and the same LIMIT (which will be the number of results per page the user wants to retrieve). The second method will be called only once and will return a list of columns for the data table. In this list, we have also added the name of the object to use as the title of our LWC (in the HTML file). The last method will be used only to determine the number of results and will be helpful to know which pagination buttons are available for the user.</p>

{% highlight java %}
public with sharing class queryClass {
@AuraEnabled(
cacheable=true
) //In this context, we are using wire services, so we need to cache our data
public static List<SObject> getResultsFromQuery(
String query,
Integer ResultsPerPage,
Integer pageNumber
) {
Integer offset = (pageNumber - 1) _ ResultsPerPage;
query += ' LIMIT ' + ResultsPerPage + ' OFFSET ' + String.valueOf(offset); //We get a simple query, and we add a limit and an offset
return Database.query(query);
}
@AuraEnabled(cacheable=true)
public static List<String> getFieldsAndObjectNamesFromSoql(String query) {
//We use this method to display all the fields on the data table
query = query.toLowerCase(query);
// We define the regex to get the name of the object
Pattern objectPattern = Pattern.compile('from\\s+([\\w\\d_]+)\\s_');
Matcher objectMatcher = objectPattern.matcher(query);
String objectName = '';
if (objectMatcher.find()) {
objectName = objectMatcher.group(1);
} else {
throw new QueryException(
'Can\'t find the object name for the query: ' + query
);
}
// We do the same for the names of the fields
Pattern fieldPattern = Pattern.compile(
'select\\s+((\\w+\\s*,\\s*)_\\w+)\\s+from\\s+' +
objectName +
'\\s_'
);
Matcher fieldMatcher = fieldPattern.matcher(query);
if (fieldMatcher.find()) {
String fieldList = fieldMatcher.group(1);
system.debug('test fieldList: ' + fieldList);
List<String> fieldNames = fieldList.split('\\s*,\\s*');

      fieldNames.add(objectName);
      for (Integer i = 0; i < fieldNames.size(); i++) {
        fieldNames[i] = fieldNames[i].capitalize(); //If we don't set the first letters as uppercase, some differences may occur between the query results and this list, and the data table will not display properly
      }
      return fieldNames;
    } else {
      throw new QueryException(
        'Can\'t find the names of the fields for the query: ' + query
      );
    }

}
//This method will be useful to know when we have to deactivate some buttons, it only returns the size of the query with no limits
@AuraEnabled(cacheable=true)
public static Integer getTotalNumberOfResults(String query) {
return Database.query(query).size();
}
}
{% endhighlight %}

<h3>Result</h3>
<p>Now, users have just to choose some results per page and a query from the app builder, and...</p>

![Result datatable app builder](/Images/datatable_app_builder.jpg)

<h3>Sources</h3>
<p>You can check these links, they are very useful:</p>
<ul>
<li><a href="https://www.lightningdesignsystem.com/components/data-tables/">Lightning Design System</a></li>
<li><a href="https://developer.salesforce.com/blogs/2014/08/paginating-data-for-force-com-applications">Pagination Salesforce Documentation</a></li>
<li><a href="https://github.com/selimhamidou/Dynamic-Datatable.git">The Github repository</a></li>
</ul>
