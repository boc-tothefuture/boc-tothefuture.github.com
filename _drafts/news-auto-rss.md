---
title:  "Automating RSS Feeds Ingest"
date:   2016-08-21 10:00:00 -0400
categories: blog bluemix Openwhisk
excerpt: "Using Openwhisk to automatically parse RSS feeds"
workflow:
  - url: news/workflow.jpg
    image_path: news/workflow.jpg
    alt: "Automatic news page workflow"
    title: "Automatic news page workflow"
---
<br>
![News](/images/news/rss.jpg)
<br>
<br>

# Automating RSS Feed Ingest with Openwhisk

This is the fifth article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and Cloudant on the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section for a blog. This post will describe how to write code for Openwhisk to automatically update the news section of a blog.

## Previous Articles

1. [Setting up Cloudant]({% post_url 2016-08-10-news-setting-up-cloudant %})
2. [Dynamic GitHub pages]({% post_url 2016-08-10-news-setting-up-cloudant %})
3. [Mixing in Openwhisk]({% post_url 2016-08-16-news-openwhisk-setup %})
3. [A unique flow in Openwhisk]({% post_url 2016-08-17-news-openwhisk-uniq %})

<br>

# Automatically Updating News page

This post will tie all the work of the previous posts together resulting in a news page that automatically updates whenever an item is posted on a blog.

{% include gallery id="workflow" caption="Automatic news page workflow" %}



## Remaining Steps

1. Create Openwhisk action to parse RSS feeds
2. Create Openwhisk action to parse article description from meta tags and invoke insert article whisk action
3. Configure a timer to periodically invoke RSS feed action

<br>

# Parsing RSS Feeds

Creating the first Openwhisk action was described in detail in [this post]({% post_url 2016-08-17-news-openwhisk-uniq %}).  In this post we will create another Openwhisk action that will trigger the insert_article action once for every article in the RSS feed.

## RSS Action

The RSS action does the following

1. Downloads and parses the RSS feed from the supplied URL.
2. Extracts the title, link and calculates a timestamp
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
            name: '/boc@us.ibm.com_blog/append_article_description',
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

A second action is used to download the article and inspect the metadata to extract the excerpt from the article.  The description supplied in the RSS item can have CDATA tags, linked to images, and other elements that are hard to parse and strip out from the text of the description.  The metadata for the article can be a more reliable, simpler source for the excerpt.


## Append Article Action Action

The append article description action does the following

1. Downloads and parses the RSS feed from the supplied URL.
2. Extracts the title, link and calculates a timestamp
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
            name: '/boc@us.ibm.com_blog/append_article_description',
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


## Create a file called append_article_description.js
{% highlight shell %}
vi append_article_description.js
{% endhighlight %}




<br>

# Executing Openwhisk action

{% highlight shell %}
./wsk action invoke --blocking insert_article --param link 'https://foo.bar/' --param title 'Openwhisk test' --param excerpt 'Testing Openwhisk' --param timestamp '1471399024'
ok: invoked /user@us.ibm.com_blog/insert_article with id 00fe0d5d634c402ca918ab273ffc8a80
{
    "namespace": "user@us.ibm.com",
    "name": "insert_article",
    "version": "0.0.1",
    "subject": "user@us.ibm.com",
    "activationId": "00fe0d5d634c402ca918ab273ffc8a80",
    "start": 1471440958662,
    "end": 1471440960573,
    "response": {
        "status": "success",
        "success": true,
        "result": {
            "msg": "Article added to Cloudant"
        }
    }
}
user@userm83:~/wsk$ ./wsk action invoke --blocking insert_article --param link 'https://foo.bar/' --param title 'Openwhisk test' --param excerpt 'Testing Openwhisk' --param timestamp '1471399024'
ok: invoked /user@us.ibm.com_blog/insert_article with id c72efe4019f8444590dbb6826a75b6e9
{
    "namespace": "user@us.ibm.com",
    "name": "insert_article",
    "version": "0.0.1",
    "subject": "user@us.ibm.com",
    "activationId": "c72efe4019f8444590dbb6826a75b6e9",
    "start": 1471440968075,
    "end": 1471440968229,
    "response": {
        "status": "success",
        "success": true,
        "result": {
            "msg": "Link already present. Ignoring article."
        }
    }
}
{% endhighlight %}

Note how the initial run has a result of "Article added to Cloudant" and the second invocation returns a result of "Link already present".

Looking at the resulting news page, only a single instance of this article has been added to the database.
{% include gallery id="no_duplicate_posts" caption="News page with no duplicate posts" %}

Now we can add entries to our Cloudant database from the command line without worrying about accidentally duplicating articles.  


# Resources
Should you run into any issues consult the [BlueMix Documentation](https://console.ng.bluemix.net/docs/), [Openwhisk Documentation](https://new-console.ng.bluemix.net/docs/openwhisk/index.html), [Cloudant Documentation](https://docs.cloudant.com/) or [reach out to me on twitter](https://twitter.com/boc_tothefuture)
