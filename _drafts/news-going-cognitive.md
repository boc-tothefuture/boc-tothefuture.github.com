---
title:  "Going Cognitive with Blog News"
date:   2016-08-27 10:00:00 -0400
categories: blog bluemix Openwhisk cognitive
excerpt: "Using Watson to provide insight into news articles"
watson_services:
  - url: news/watson_services.jpg
    image_path: news/watson_services.jpg
    alt: "Watson services on Bluemix"
    title: "Watson services on Bluemix"
watson_credentials:
  - url: news/watson_credentials.jpg
    image_path: news/watson_credentials.jpg
    alt: "Alchemy API credentials on Bluemix"
    title: "Alchemy API credentials on Bluemix"
cloudant_watson_document:
  - url: news/cloudant_watson_document.jpg
    image_path: news/cloudant_watson_document.jpg
    alt: "Cloudant document with Watson additions"
    title: "Cloudant document with Watson additions"
---
<br>
![News](/images/news/watson.jpg)
<br>
<br>

# Watson Cognitive

This is the sixth article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and Cloudant on the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section for a blog.

## Previous Articles

1. [Setting up Cloudant]({% post_url 2016-08-10-news-setting-up-cloudant %})
2. [Dynamic GitHub pages]({% post_url 2016-08-10-news-setting-up-cloudant %})
3. [Mixing in Openwhisk]({% post_url 2016-08-16-news-openwhisk-setup %})
4. [A unique flow in Openwhisk]({% post_url 2016-08-17-news-openwhisk-uniq %})
5. [Automating RSS Feed Ingest]({% post_url 2016-08-24-news-auto-rss %})


This post will describe how to use the cognitive technologies of Watson with Openwhisk to automatically provide insight into the articles on a blog news page.

## Steps

1. Add Alchemy API to your Bluemix account
2. Insert a new Openwhisk action in the news chain to append cognitive insights on news articles
3. Update front end code to display cognitive insights

<br>

# Add Alchemy API service to your Bluemix account

Login into your BlueMix account [here](https://console.ng.bluemix.net/).  After you login, select catalog from the top navigation. On the left navigation bar select the checkbox next to Watson to limit the display to Watson services.  Your screen should look like the one below.

{% include gallery id="watson_services" caption="List of Watson services on Bluemix" %}

Select Alchemy API, leave the defaults values and click Create.
After a few moments, the Watson Alchemy API service should be created and added to your Bluemix space.  

Click on Service credentials on the left navigation and note your API key.

{% include gallery id="watson_credentials" caption="Watson credentials" %}

That is it!  Your Bluemix space is now Watson enabled.

# Watson Openwhisk Action


## Create a new Openwhisk action
This Openwhisk action the following

1. Sends a URL to the Alchemy API service
2. Extract keywords, concepts, and document emotion
3. Appends watson insights to the article information
3. Invokes the insert action to insert the article

The contents of the append_watson action in its entirety:

{% highlight javascript %}
//Access watson APIs
var watson = require('watson-developer-cloud');

function main(params) {

// Set the API Key from our Bluemix portal
var alchemy_language = watson.alchemy_language({
  api_key: '<YOUR KEY HERE>'
});

var parameters = {
  // Extract keywords, concepts, and document emotion
  extract: 'keywords,concepts,doc-sentiment',
  //enable targeted sentiment information for entities and keywords
  sentiment: 1,
  //Max number of concepts to retrieve
  maxRetrieve: 8,
  //Link to process
  url: params.link
};


return new Promise(function(resolve, reject) {
  alchemy_language.combined(parameters, function (err, response) {
    if (err) {
      reject(err);
    }
    else {
      //Add alchemy data to parameters
      params.sentiment = response.docSentiment;
      params.keywords =  response.keywords;
      params.concepts =  response.concepts;
      //Invoke the insert article action
      whisk.invoke({
        name: '/user@us.ibm.com_blog/insert_article',
        parameters: params
      });
      resolve({msg: 'done'});
    }
  });
});

}

{% endhighlight %}

## Modify append article description action
Previously, the append article description action invoked the insert article action.  To insert the append watson service into our chain of action we need to swap out the insert artcle whisk call for the append watson call.  

Just change this
{% highlight javascript %}
//Invoke the insert article action
  whisk.invoke({
    name: '/user@us.ibm.com_blog/insert_article',
    parameters:{
      "title": params.title,
      "link": params.link,
      "excerpt": excerpt,
      "timestamp": String(params.timestamp)
    }
  });
{% endhighlight %}

to this

{% highlight javascript %}
//Invoke the append watson action
  whisk.invoke({
    name: '/user@us.ibm.com_blog/append_watson',
    parameters:{
      "title": params.title,
      "link": params.link,
      "excerpt": excerpt,
      "timestamp": String(params.timestamp)
    }
  });
{% endhighlight %}

## Modify insert article
Previously, the insert article method was prescriptive in what parameters it put into the Cloudant doc.  To provide for more flexibility, we are now opening that up so that all parameters are now part of the document stored in Cloudant.

The updated Openwhisk action just passes the params object as the value for 'doc'
{% highlight javascript %}
whisk.invoke({
  name: '/user@us.ibm.com_blog/blog_news/write',
   parameters:{
     dbname: 'blog_news',
     doc: params},
   blocking: true,
   next:  function(error, activation) {
     console.log('error:', error, 'activation:', activation);
     if(error){
       whisk.error(error);
     }
     else{
       whisk.done({msg:'Article added to Cloudant'});
     }
   }
});

{% endhighlight %}


Once you update all your Openwhisk actions, your cloudants documents should have a lot more information.

{% include gallery id="cloudant_watson_document" caption="Cloudant document with Watson information added" %}

# Display cognitive insights

A second action is used to download the article and inspect the metadata to extract the excerpt from the article.

You may be wondering?

> Why don't we just use the description filed in the item RSS?

Because The description supplied in the RSS item can have CDATA tags, linked to images, and other elements that are hard to parse and strip out from the text of the description.  The metadata for the article is a more reliable simpler source for the excerpt.


## Append Article Description Action Action

The append article description action does the following

1. Downloads the article from the supplied link
2. Determines the excerpt using an ordered priority of meta data
  * [OpenGraph](http://ogp.me/) Description
  * [Twitter](https://dev.twitter.com/cards/markup) Description
  * [HTML](http://www.w3schools.com/tags/tag_meta.asp) Description
3. If at least one of those can be resolved, then invoke the [insert article]({% post_url 2016-08-17-news-openwhisk-uniq %}) whisk action.

The contents of the append_article_description action in its entirety:

{% highlight javascript %}
var url = params.link;

 return new Promise(function(resolve, reject) {
   request.get(url, function(error, response, body) {
     if (error) {
       reject(error);
     }
     else {
       //Parse the body of the article
       var $ = cheerio.load(body);

       //Extract the OpenGraph description
       var excerpt = $('meta[property="og:description"]').attr('content');
       if( excerpt == null) {
         //Extract the twitter if no OG description
         excerpt = $('meta[name="twitter:description"]').attr('content');
         if( excerpt == null) {
           //Fall back to the HTML meta description
           excerpt = $('meta[name="description"]').attr('content');
         }
       }

       if( excerpt ) {
         //Parse the excerpt using cheerio, returning HTML represetation of excerpt.
         //The html method is needed to properly escape any HTML coded items in the
         //description
         excerpt = cheerio.load(excerpt).html();

         //Invoke the insert article action
         whisk.invoke({
           name: '/user@us.ibm.com_blog/insert_article',
           parameters:{
             "title": params.title,
             "link": params.link,
             "excerpt": excerpt,
             "timestamp": String(params.timestamp)
           }
         });
       }
       else{
         reject({msg: 'Could not resolve article excerpt'});
       }
       resolve({msg: 'done'});
     }
   });
 });

{% endhighlight %}

<br>

# Manually invoke news feed update

Invoke the created rss action, passing it a URL of a news feed to parse.

In the example below the news feed for the [Chef Blog](https://blog.chef.io/) is used.
{% highlight shell %}
./wsk action invoke --blocking rss --param url 'https://blog.chef.io/feed/'
ok: invoked /user@us.ibm.com_blog/rss with id 219a76cf92aa40b5bad643e2e70088dd
{
    "namespace": "user@us.ibm.com",
    "name": "rss",
    "version": "0.0.105",
    "subject": "user@us.ibm.com",
    "activationId": "219a76cf92aa40b5bad643e2e70088dd",
    "start": 1472000143923,
    "end": 1472000144623,
    "response": {
        "status": "success",
        "success": true,
        "result": {
            "msg": "done"
        }
    }
}

{% endhighlight %}

After that executes, the news page should be automatically updated with all Chef news feed items.

{% include gallery id="chef_news" caption="Automatic news page with Chef news items added" %}


# Automatically updating the blog news feed.

To have a periodically automatically updating news feed two more steps are required.

1. Create a trigger that fires periodically
2. Map that trigger to the rss action defined above using an Openwhisk rule

## Periodic Trigger

To periodicallly trigger an action we will use the Openwhisk [Alarm Trigger](https://console.ng.bluemix.net/docs/openwhisk/openwhisk_catalog.html#openwhisk_catalog_alarm) in [Bluemix](http://www.ibm.com/bluemix).

The following creates a trigger called chef_news_trigger that fires once a day at 8 am.  The trigger_payload will pass the Chef Blog RSS URL to the action associated with the job.
{% highlight shell %}
./wsk trigger create chef_news_trigger --feed /whisk.system/alarms/alarm --param cron '0 0 8 * * *' --param trigger_payload '{"url":"https://blog.chef.io/feed/"}'
ok: invoked /whisk.system/alarms/alarm with id 11f07fc6ec7646b5924d3ab4fad8909d
{
    "namespace": "user@us.ibm.com",
    "name": "alarm",
    "version": "0.0.61",
    "subject": "user@us.ibm.com",
    "activationId": "11f07fc6ec7646b5924d3ab4fad8909d",
    "start": 1472001489670,
    "end": 1472001489809,
    "response": {
        "status": "success",
        "success": true
    }
}
ok: created trigger chef_news_trigger
{% endhighlight %}

## Trigger mapping

The trigger defined above will run once a day.  The next step is to map what happens when that trigger fires.  To create that mapping a [Openwhisk Rules](https://console.ng.bluemix.net/docs/openwhisk/openwhisk_triggers_rules.html#openwhisk_rules_use) are used.

The following creates a rule called the chef_news_rule that maps a firing of the chef_news_trigger to the rss action.
{% highlight shell %}
./wsk rule create --enable chef_news_rule chef_news_trigger rss
ok: created rule chef_news_rule
{% endhighlight %}

With that rule created, the next time that trigger fires the news section will be automatically updated.

For this blog, I want a variety of news sources, so I will add more rules.  I am adding [ansible](https://www.ansible.com/) and [puppet](https://puppet.com/) news feeds.

I do spread out the trigger time to update the news sources throughout the day, advancing each trigger by an hour.

{% highlight shell %}
./wsk trigger create ansible_news_trigger --feed /whisk.system/alarms/alarm --param cron '0 0 2 * * *' --param trigger_payload '{"url":"https://www.ansible.com/blog/rss.xml"}'
ok: invoked /whisk.system/alarms/alarm with id 546e810a4ea6496e813cb4e7a6246fbf
{
   "namespace": "user@us.ibm.com",
   "name": "alarm",
   "version": "0.0.61",
   "subject": "user@us.ibm.com",
   "activationId": "546e810a4ea6496e813cb4e7a6246fbf",
   "start": 1472002206314,
   "end": 1472002206351,
   "response": {
       "status": "success",
       "success": true
   }
}
ok: created trigger ansible_news_trigger
user@userm83:~/wsk$ ./wsk rule create --enable ansible_news_rule ansible_news_trigger rss
ok: created rule ansible_news_rule
user@userm83:~/wsk$ ./wsk trigger create puppet_news_trigger --feed /whisk.system/alarms/alarm --param cron '0 0 3 * * *' --param trigger_payload '{"url":"https://puppet.com/blog/rss.xml"}'
ok: invoked /whisk.system/alarms/alarm with id e42f819d323a49deb57b9c0d204276dd
{
   "namespace": "user@us.ibm.com",
   "name": "alarm",
   "version": "0.0.61",
   "subject": "user@us.ibm.com",
   "activationId": "e42f819d323a49deb57b9c0d204276dd",
   "start": 1472002262509,
   "end": 1472002262545,
   "response": {
       "status": "success",
       "success": true
   }
}
ok: created trigger puppet_news_trigger
user@userm83:~/wsk$ ./wsk rule create --enable puppet_news_rule puppet_news_trigger rss      ok: created rule puppet_news_rule
{% endhighlight %}


<br>

After about a day the news page should be fully populated by a variety or sources.

{% include gallery id="multiple_news" caption="Automatic news page with multiple sources" %}

Enjoy your automatically updating serverless news page!

# Resources
Should you run into any issues consult the [BlueMix Documentation](https://console.ng.bluemix.net/docs/), [Openwhisk Documentation](https://new-console.ng.bluemix.net/docs/openwhisk/index.html), [Cloudant Documentation](https://docs.cloudant.com/) or [reach out to me on twitter](https://twitter.com/boc_tothefuture)
