---
title:  "Cloudant for news"
date:   2016-08-13 13:40:00 -0400
categories: blog cloudant bluemix
excerpt: "Use Cloudant in Bluemix for dynamic news section of a blog"
cloudant_cors:
  - url: news/cloudant_cors.jpg
    image_path: news/cloudant_cors.jpg
    alt: "CORS page for Cloudant"
    title: "Default CORS page for Cloudant"
cors_added_domain:
  - url: news/cors_added_domain.jpg
    image_path: news/cors_added_domain.jpg
    alt: "CORS page for Cloudant with domain added"
    title: "CORS page with domain added"
cloudant_index:
  - url: news/cloudant_index.jpg
    image_path: news/cloudant_index.jpg
    alt: "Create a Cloudant index"
    title: "Create a Cloudant index"
cloudant_create_index:
  - url: news/cloudant_create_index.jpg
    image_path: news/cloudant_create_index.jpg
    alt: "Create the timestamp index"
    title: "Create a timestamp index"

---
<br>
![News](/images/news/github_logo.png)
<br>
<br>

# Creating an automatically updating news page

This is the second article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section.  The first article in this series is located [here]({% post_url 2016-08-10-news-setting-up-cloudant %}).

In this article we will
1. Setup Cross origin Resource Sharing
2. Create a Cloudant index
3. Build a blog page to access and render those news articles


# Cross Origin Resource Sharing (CORS)

Because I am consuming the [Cloudant](http://www.ibm.com/Cloudant) [Database as a Service](https://en.wikipedia.org/wiki/Cloud_database) I need to perform a few steps to authorize a web browser to access data from Cloudant  which is served off a different domain name.  Cloudant uses a technology called [Cross Origin Resource Sharing](https://en.wikipedia.org/wiki/Cross-origin_resource_sharing) or CORS to enable me to access from Cloudant from my blog. The cloudant [docs for CORS](https://docs.cloudant.com/cors.html) cover in detail how to configure CORS for a variety of usage scenarios.

## Setting up CORS on Cloudant
Launch your Cloudant Dashboard from your BlueMix account.  Once launched, select Account from the left navigation section and then select CORS from the slide out menu.  
{% include gallery id="cloudant_cors" caption="Default CORS setup in Cloudant" %}

By default, CORS is enabled by no domains are specified.  I am going to add the domains of this blog in the provided text box.
If all goes well, you should your blogs domain now authorized for CORS requests.
{% include gallery id="cors_added_domain" caption="CORS with domain in Cloudant" %}

# Create a Cloudant index
While still in Cloudant, select Databases from the left navigation section and then select your blog_news database. Select the plus sign next to Design Documents then select query index.
{% include gallery id="cloudant_index" caption="Create a new query index" %}

In the next screen you can create your index.  In our example we want to index the timestamp of the news articles so we can display the last X number of articles. All that has to be done is to
replace 'foo' with 'timestamp' and click create index.
{% include gallery id="cloudant_create_index" caption="Index creation" %}

With the index created, we can now query Cloudant for our news articles.


# Access Cloudant data from GitHub pages

To create our dynamic news page served by GitHub pages we will need to do a few things.

* Create a new top level news page in your [Jekyll](https://jekyllrb.com/) based blog
* Craft some HTML and Javascript to dynamically render the news items

## Create a top level news page
I am using [minimal mistakes](https://mmistakes.github.io/minimal-mistakes) so I am following the instructions [here](https://mmistakes.github.io/minimal-mistakes/docs/pages/) to create a new top level page for our blog.
{% highlight shell %}
cd _pages
vi news.html
{% endhighlight %}

I add the following YAML front matter to my page.
{% highlight yml %}
---
title: "Infrastruture DevOps News"
layout: single
permalink: /news/
author_profile: true
custom_js:
- jquery-1.12.4.min
---
{% endhighlight %}

You may have noticed the custom_js field in the YAML.  We added that so we can load jquery into our and use jquery to access Cloudant.
In order, to make this work we need to put jquery-1.12.4.min into a 'js' directory in our blog and then update the head.html
to load this javascript on pages that need it.  

See [this page](http://mattgemmell.com/page-specific-assets-with-jekyll/) by Matt GemmeLL's for info on the code to put into your head.html


Add your new page to the navigation.yml in the _data directory. After adding news my YAML looked like this:

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



# Resources
Should you run into any issues consult the [BlueMix Documentation](https://console.ng.bluemix.net/docs/), the [Cloudant Documentation](https://docs.cloudant.com/) or [reach out to me on twitter](https://twitter.com/boc_tothefuture)
