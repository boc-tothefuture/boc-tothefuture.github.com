---
title:  "Dynamic GitHub Pages"
date:   2016-08-15 12:00:00 -0400
categories: blog cloudant bluemix
excerpt: "Use Cloudant in Bluemix for dynamic news section of a GitHub pages Blog"
cloudant_cors:
  - url: news/cloudant_cors.jpg
    image_path: images/news/cloudant_cors.jpg
    alt: "CORS page for Cloudant"
    title: "Default CORS page for Cloudant"
cors_added_domain:
  - url: news/cors_added_domain.jpg
    image_path: images/news/cors_added_domain.jpg
    alt: "CORS page for Cloudant with domain added"
    title: "CORS page with domain added"
cloudant_index:
  - url: news/cloudant_index.jpg
    image_path: images/news/cloudant_index.jpg
    alt: "Create a Cloudant index"
    title: "Create a Cloudant index"
cloudant_create_index:
  - url: news/cloudant_create_index.jpg
    image_path: images/news/cloudant_create_index.jpg
    alt: "Create the timestamp index"
    title: "Create a timestamp index"
news_page:
  - url: news/news_page.jpg
    image_path: images/news/news_page.jpg
    alt: "News page"
    title: "News page"
---
<br>
![News](/images/news/github_logo.png)
<br>
<br>

# Creating an automatically updating news page

This is the second article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and Cloudant on the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section for a blog.  The first article in this series is located [here]({% post_url 2016-08-10-news-setting-up-cloudant %}).

In this post we will

1. Setup Cross Origin Resource Sharing
2. Create a Cloudant index
3. Build a blog page to access and render those news articles


# Cross Origin Resource Sharing (CORS)

Consuming the [Cloudant](http://www.ibm.com/Cloudant) [Database as a Service](https://en.wikipedia.org/wiki/Cloud_database) from [GitHub Pages](https://pages.github.com/) requires a few steps to authorize a web browser to access data from Cloudant  which is served off a different domain name.  Cloudant uses a technology called [Cross Origin Resource Sharing](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) or CORS to enable access to Cloudant from my blog. The Cloudant [docs for CORS](https://docs.cloudant.com/cors.html) cover in detail how to configure CORS for a variety of usage scenarios.

## Setting up CORS on Cloudant
Launch Cloudant Dashboard from your BlueMix account.  Once launched, select Account from the left navigation section and then select CORS from the slide out menu.  
{% include gallery id="cloudant_cors" caption="Default CORS setup in Cloudant" %}

By default, CORS is enabled but no domains are specified.  Add the domains for your blog in the provided text box.
Your blogs domain now authorized for CORS requests.
{% include gallery id="cors_added_domain" caption="CORS with domain in Cloudant" %}

# Create a Cloudant index
While still in Cloudant, select Databases from the left navigation section and then select your blog news database. Select the plus sign next to Design Documents then select query index.
{% include gallery id="cloudant_index" caption="Create a new query index" %}

In the next screen you can create your index.  In our example we want to index the timestamp of the news articles so we can sort and display the last X number of articles. From the supplied default index replace 'foo' with 'timestamp' and click create index.
{% include gallery id="cloudant_create_index" caption="Index creation" %}

With the index created, Cloudant can now be queried for news articles.


# Access Cloudant data from GitHub pages

To create our dynamic news page served by GitHub pages we will need to do a few things.

* Create a new top level news page in your [Jekyll](https://jekyllrb.com/) based blog
* Craft some HTML and Javascript to dynamically render the news items

## Create a top level news page
I am using [minimal mistakes](https://mmistakes.github.io/minimal-mistakes) so I am following the instructions [here](https://mmistakes.github.io/minimal-mistakes/docs/pages/) to create a new top level page for my blog.

{% highlight shell %}
cd _pages
vi news.html
{% endhighlight %}

Add the following YAML front matter to the news page.
{% highlight yml %}
title: "Infrastruture DevOps News"
layout: single
permalink: /news/
author_profile: true
custom_js:
- jquery-1.12.4.min
- jquery-dateFormat.min
{% endhighlight %}

Note the custom_js field in the YAML.  That is inserted to load [jQuery](http://api.jquery.com/) to access to Cloudant.  jquery-1.12.4.min and jquery-dateFormat.min are downloaded into a 'js' directory in the git repo for the blog.  The head.html for all pages is modified to load this javascript only on pages that require it. See [this page](http://mattgemmell.com/page-specific-assets-with-jekyll/) by Matt Gemmell's for info on the code to put into your head.html


Add your new page to the navigation.yml in the _data directory. After adding news, my YAML looked like this:

{% highlight yml %}
main:
  - title: "Categories"
    url: /categories/

  - title: "News"
    url: /news/

  - title: "About"
    url: /about/
{% endhighlight %}

## Access Cloudant from HTML/Javascript

Here is the content of the news page in its entirety.

{% highlight html %}
<br>
<div id="result"></div>

<script>

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
    "excerpt"
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
        $( "#result" ).append( "<i>" +  DateFormat.format.prettyDate(new Date(1000*value.timestamp).getTime(), "dd/MM/yyyy") + "</i>" );
        $('<br>').appendTo("#result");
        $('<br>').appendTo("#result");
      });
  }
});

</script>
{% endhighlight %}

There is a lot going on.  So, lets break it down by section.

### HTML Section

The pure HTML component of this page consists of creating a blank line and creating a div for the script code to update later.

{% highlight html %}
<br>
<div id="result"></div>
{% endhighlight %}

### Query Section

This is the query we are sending to Cloudant.  The query consists of a selector, fields and sort section.  The selector tells Cloudant to pull documents that have a
timestamp field that is greater than zero.  The fields section instructs Cloudant to return these specified fields in the result.  Finally, the sort section tells
Cloudant to sort the results by the timestamp field in descending order.

{% highlight html %}
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
    "excerpt"
  ],
  "sort": [
    {
      "timestamp": "desc"
    }
  ]
};
{% endhighlight %}

### Cloudant Request Section
This generates the Cloudant request and updates the web page with the results.  The request is executed using the [jQuery ajax function](http://api.jquery.com/jquery.ajax/).

{% highlight html %}
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
        $( "#result" ).append( "<i>" +  DateFormat.format.prettyDate(new Date(1000*value.timestamp).getTime()) + "</i>" );
        $('<br>').appendTo("#result");
        $('<br>').appendTo("#result");
      });
  }
});
{% endhighlight %}

Lets break down all parts of the AJAX request.

#### URL
The URL for the request is taken from your Cloudant instance.  The format is <base_cloudant_url>/<database>/_find?limit=<N>.  The limit is optional.  In my case I only want to show the latest 10 news items, so I include the limit query parameter with a value of 10.

#### Authorization
The data in the authorization header is the base64 encoded Key and Password you created [in part 1]({% post_url 2016-08-10-news-setting-up-cloudant %}).  To generate this value you take the key and password, separate them with a ':' and then encode them in base64.  I used [this online encoder](http://www.tuxgraphics.org/toolbox/base64-javascript.html) for encoding.

#### Data
The data section takes the query object we created earlier and turns that into a JSON string to send as the query to the Cloudant server.

#### Function
The success function is run after the Cloudant returns results for the query.  Those results are processed through several steps.

* Empty the results div we created earlier
* Iterate over all returned documents
* Add a link with The link text is the title of the article stored in Cloudant and the href if the link to the article
* Add a blank line
* Add the excerpt of the news article from the Cloudant document
* Add a blank line
* In italics, pretty format the date of when the article was created (This uses [jQuery date format](https://github.com/phstc/jquery-dateFormat)).
* Add in two blank lines

After you push this up to [GitHub Pages](https://pages.github.com/) you should have an automatically updated news page like this!
{% include gallery id="news_page" caption="Example automatically generated news page" %}


# Resources
Should you run into any issues consult the [BlueMix Documentation](https://console.ng.bluemix.net/docs/), the [Cloudant Documentation](https://docs.cloudant.com/) or [reach out to me on twitter](https://twitter.com/boc_tothefuture)
