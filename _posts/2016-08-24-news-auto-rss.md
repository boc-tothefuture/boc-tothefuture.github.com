---
title:  "Automating RSS Feeds Ingest"
date:   2016-08-24 10:00:00 -0400
categories: blog bluemix Openwhisk chef puppet ansible
excerpt: "Using Openwhisk to automatically parse RSS feeds"
workflow:
  - url: news/workflow.jpg
    image_path: images/news/workflow.jpg
    alt: "Automatic news page workflow"
    title: "Automatic news page workflow"
chef_news:
  - url: news/chef_news.jpg
    image_path: images/news/chef_news.jpg
    alt: "Automatic news page with Chef blog items added"
    title: "Automatic news page workflow with Chef blog items added"
multiple_news:
  - url: news/multiple_news.jpg
    image_path: images/news/multiple_news.jpg
    alt: "Automatic news page with multiple news sources"
    title: "Automatic news page workflow with multiple news sources"
---
<br>
![News](/images/news/rss.jpg)
<br>
<br>

# RSS Feed Ingest with Openwhisk

This is the fifth article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and Cloudant on the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section for a blog.

This post will describe how to write code for Openwhisk to automatically update the news section of a blog.

## Previous Articles

1. [Setting up Cloudant]({% post_url 2016-08-10-news-setting-up-cloudant %})
2. [Dynamic GitHub pages]({% post_url 2016-08-10-news-setting-up-cloudant %})
3. [Mixing in Openwhisk]({% post_url 2016-08-16-news-openwhisk-setup %})
4. [A unique flow in Openwhisk]({% post_url 2016-08-17-news-openwhisk-uniq %})

<br>

# Automatically Updating News page

This post will tie all the work of the previous posts together resulting in a news page that automatically updates whenever an item is posted on a blog.

{% include gallery id="workflow" caption="Automatic news page workflow" %}



## Remaining Steps

1. Create Openwhisk action to parse RSS feeds
2. Create Openwhisk action to extract article description from meta tags and invoke insert article whisk action
3. Configure an Openwhisk trigger to periodically invoke RSS feed action

<br>

# Parsing RSS Feeds

Creating the first Openwhisk action was described in detail in [this post]({% post_url 2016-08-17-news-openwhisk-uniq %}).  In this post we will create another Openwhisk action that will invoke the append_article_description action once for every article in the RSS feed.

## RSS Action

The RSS action does the following

1. Downloads and parses the RSS feed from the URL supplied as a parameter
2. Extracts the title, link and calculates a timestamp since epoch
3. Invokes the append_article_description action to create an article extract

The contents of the rss action in its entirety:

{% highlight javascript %}
// Request module is used to download RSS feed
var request = require('request');

// Cheerio module is used to parse the XML in the RSS feed.
var cheerio = require("cheerio");

function main(params) {
  var url = params.url;

  return new Promise(function(resolve, reject) {
    request.get(url, function(error, response, body) {
      if (error) {
        reject(error);
      }
      else {
        // Parse the RSS Feed XML provided as a response body
        $ = cheerio.load(body, {
          normalizeWhitespace: true,
          xmlMode: true
        });

        // Iterate over all RSS items in the feed.
        $( "item" ).each(function(){
          // Extract the title and link from the item in the RSS feed.
          var title = $(this).find('title').text();
          var link = $(this).find('link').text();
          // Convert the item pubDate then calculate the timestamp in seconds since epoch.
          var timestamp = Date.parse($(this).find('pubDate').text())/1000;

          // Pass the parsed data onto append_article_description action
          whisk.invoke({
            name: '/user@us.ibm.com_blog/append_article_description',
            parameters:{
              "title": title,
              "link": link,
              "timestamp": timestamp
            }
          });

        });
        resolve({msg: 'done'});
      }
    });
  });

}
{% endhighlight %}

# Append article description

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
