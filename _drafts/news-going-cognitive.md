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
cognitive_news:
  - url: news/cognitive_news.jpg
    image_path: news/cognitive_news.jpg
    alt: "Blog news with cognitive insights"
    title: "Blog news with cognitive insights"
---

<br>
![News](/images/news/watson.jpg)
<br>
<br>

# Watson Cognitive

This is the sixth article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and [Cloudant](https://cloudant.com/) on the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section for a blog.

## Previous Articles

1. [Setting up Cloudant]({% post_url 2016-08-10-news-setting-up-cloudant %})
2. [Dynamic GitHub pages]({% post_url 2016-08-10-news-setting-up-cloudant %})
3. [Mixing in Openwhisk]({% post_url 2016-08-16-news-openwhisk-setup %})
4. [A unique flow in Openwhisk]({% post_url 2016-08-17-news-openwhisk-uniq %})
5. [Automating RSS Feed Ingest]({% post_url 2016-08-24-news-auto-rss %})

<br>

This post describes how to use Openwhisk to access the cognitive technologies of Watson to automatically provide insights into the articles on a blog news page.

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

Click on service credentials on the left navigation and note your API key.

{% include gallery id="watson_credentials" caption="Watson credentials" %}

Your Bluemix space is now Watson enabled.

# Watson Openwhisk Action


## Create a new Openwhisk action
This Openwhisk action does the following

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

The news page that was created [here]({% post_url 2016-08-15-news-dynamic-github-pages %}) is updated to display the Watson data now inserted into Cloudant.  Below is the updated javascript necessary to display the cognitive news section.

{% highlight javascript %}
var query = {
  "selector": {
    "timestamp": {
      "$gt": 0
    }
  },
  "fields": [
    "title",
    "link",
    "timestamp",
    "excerpt",
    "sentiment",
    "keywords",
    "concepts"
  ],
  "sort": [
    {
      "timestamp": "desc"
    }
  ]
};

$.ajax({
  type: "POST",
  url: "https://2619ca16-f337-42b9-b353-08009fcf31d7-bluemix.cloudant.com/blog_news/_find?limit=10",
  headers: {"Authorization": "Basic aWNoZXR0ZWVkaWVyaW5nZXNvbWVuZXdoOmVjNGI3OWY0MDJmYzhmYzljNTY3MGI5NmVmMTM1OWMyZDgzNDJhNmM="},
  data: JSON.stringify( query ),
  contentType: "application/json; charset=utf-8",
  dataType: "json",
  success: function(data) {
      $( "#result" ).empty();
      $(data.docs).each(function (index, value) {
        $('<a>',{
          text: value.title,
          href: value.link
        }).appendTo( "#result" );
        $('<br>').appendTo("#result");
        $( "#result" ).append( value.excerpt );
        $('<br>').appendTo("#result");

        concepts = jQuery.map( value.concepts, function(concept, index ){
          return concept.text;
        });

        keywords = jQuery.map( value.keywords, function(keyword, index ){
          return keyword.text;
        });

        var elements = { 'Published' : DateFormat.format.prettyDate(new Date(1000*value.timestamp).getTime()),
                         'Sentiment' : '<img src="/images/news/' + value.sentiment.type + '.png">',
                         'Concepts' : concepts.join(", "),
                         'Keywords' : keywords.join(", ")
                       };

        var $table = $('<table/>');
        $table.css('border-collapse','collapse');
        $table.css('border','none');
        jQuery.each(elements, function (name, value) {
          $table.append('<tr><td style="border: none;">' + name + '</td><td style="border: none;">' + value + '</td></tr>');
        });

        $( "#result" ).append( $table );

        $('<br>').appendTo("#result");
      });
  }
});

{% endhighlight %}

A few changes of note in the updated javascript.  

* Cloudant is asked to return three additional fields: Sentiment, Concepts, Keywords
* A dynamic table object without borders is created and inserted using [jQuery](https://jquery.com/)

After those changes, the news page should look like the one below.

{% include gallery id="cognitive_news" caption="News with cognitive insights" %}


That is it!  Using Bluemix with Watson services it's easy to add cognitive insight into your blog.  All without managing a single server.

# Resources
Should you run into any issues consult the [BlueMix Documentation](https://console.ng.bluemix.net/docs/), [Openwhisk Documentation](https://new-console.ng.bluemix.net/docs/openwhisk/index.html), [Cloudant Documentation](https://docs.cloudant.com/) or [reach out to me on twitter](https://twitter.com/boc_tothefuture)
