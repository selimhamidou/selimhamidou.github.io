---
layout: post
title:  "sfdx Survival book"
date:   2023-03-28 19:49:26 +0200
categories: jekyll update
---
This post is not finished. At this time, I am using a lot sfdx, and instead of searching for the good informations everywhere in the web, I am just trying to get everything I need on the same place. So, I will begin with a couple of commands, and I will add some others, from time to time. 

<h3>Create a project and authorize an org</h3>
{% highlight shell %}
sfdx force:project:create -n MyProject
sfdx auth:web:login --setalias my-hub-org --instanceurl https://test.salesforce.com
{% endhighlight %}

<h3>Retrieve components from a package</h3>
{% highlight shell %}
sfdx force:source:retrieve -p path/to/source
sfdx force:source:retrieve -m ApexClass     //Retrieve all apex classes
sfdx force:source:retrieve -m ApexClass:MyApexClass     //Retrieve a specific apex class
sfdx force:source:retrieve -x path/to/package.xml       //Retrieve components from package
{% endhighlight %}

<h3>Deploy components from a package</h3>
{% highlight shell %}
sfdx force:source:deploy --manifest path/to/manifest/manifest.xml -l RunSpecifiedTests -r TestClass1,TestClass2 -w 33 --verbose --loglevel fatal -u username
{% endhighlight %}

<h3>Run apex tests</h3>
{% highlight shell %}
sfdx force:apex:test:run --classnames "TestA,TestB" --resultformat tap --codecoverage
sfdx force:apex:test:run --tests "ns.TestA.excitingMethod,ns.TestA.boringMethod,ns.TestB"
{% endhighlight %}
