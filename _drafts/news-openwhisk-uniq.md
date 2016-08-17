---
title:  "A unique flow in Openwhisk"
date:   2016-08-16 12:00:00 -0400
categories: blog cloudant bluemix Openwhisk
excerpt: "Creating a unique news posts in Cloudant with Openwhisk"
duplicate_posts:
  - url: news/duplicate_posts.jpg
    image_path: news/duplicate_posts.jpg
    alt: "Duplicate news posts"
    title: "Duplicate news posts"
cloudant_index:
  - url: news/cloudant_index.jpg
    image_path: news/cloudant_index.jpg
    alt: "Create a Cloudant index"
    title: "Create a Cloudant index"
cloudant_create_index:
  - url: news/cloudant_create_link_index.jpg
    image_path: news/cloudant_create_link_index.jpg
    alt: "Create the link index"
    title: "Create a link index"  
---
<br>
![News](/images/news/unique.jpg)
<br>
<br>

# Ensuring unique news articles

This is the forth article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and Cloudant on the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section for a blog. This post will describe how to configure your Openwhisk chain to make sure news items on your blog are only posted once.

## Previous Articles

1. [Setting up Cloudant]({% post_url 2016-08-10-news-setting-up-cloudant %}).
2. [Dynamic GitHub pages]({% post_url 2016-08-10-news-setting-up-cloudant %}).
3. [Mixing in Openwhisk]({% post_url 2016-08-16-news-openwhisk-setup %}).

<br>

# Duplicate article problem

The [last article]({% post_url 2016-08-16-news-openwhisk-setup %}) of this series explained how to configure Openwhisk to let us update news articles from the command line.  The problem with the
simple approach outlined in that post is that the same news item could show up multiple times in the news page.

Look what happens when we call the same Openwhisk action twice with the same values.
{% highlight shell %}
./wsk action invoke /user@us.ibm.com_blog/blog_news/write --blocking --result --param dbname blog_news --param doc '{"title":"news 3", "link":"http://www.example.com/news3", "excerpt":"Example", "timestamp":"1471314119"}
{% endhighlight %}

{% highlight javascript %}
{

    "id": "62a685b09723df4dd6759443f65a2389",
    "ok": true,
    "rev": "1-a0ac96533ae7991b23ea2f4d76d48b6b"
}
{% endhighlight %}

{% highlight shell %}
./wsk action invoke /user@us.ibm.com_blog/blog_news/write --blocking --result --param dbname blog_news --param doc '{"title":"news 3", "link":"http://www.example.com/news3", "excerpt":"Example", "timestamp":"1471314119"}
{% endhighlight %}

{% highlight javascript %}
{
  "id": "feb502e2f95b7462a805b1459972c1b4",
  "ok": true,
  "rev": "1-a0ac96533ae7991b23ea2f4d76d48b6b"  
}
{% endhighlight %}

{% include gallery id="duplicate_posts" caption="News page with duplicate posts" %}


## Steps to ensure unique articles

1. Create a Cloudant index on the link field.
2. Filter news articles with links that already exist in Cloudant
3. Insert filtered news articles.

<br>

# Another Cloudant index

Creating the initial Cloudant index was described in detail in [this post]({% post_url 2016-08-10-news-setting-up-cloudant %}).  To create another index open up the Cloudant portal, select Databases from the left navigation section and then select your blog news database. Select the plus sign next to Design Documents then select query index.
{% include gallery id="cloudant_index" caption="Create a new query index" %}

In the next screen you can create your index.  Unlike the previous index, this time we want to index the link of the news articles so we can make sure all news artciles are unique. From the supplied default index example in cloudant replace 'foo' with 'link' and click create index.
{% include gallery id="cloudant_create_index" caption="Link index creation" %}

With the index created, Cloudant can now be quickly queried to see if a link already exists.

# Filter news articles
A custom Openwhisk action will be used to filter news articles to ensure they are unique.  This filter will be written in Javascript.

Create a file called blog_news.js
{% highlight shell %}
vi blog_news.js
{% endhighlight %}

Here is the content of the blog_news action in its entirety
{% highlight javascript %}

{% endhighlight %}

Create and upload the action to Openwhisk
{% highlight shell %}
./wsk action create blog_news_input blog_news.js
ok: created action blog_news_input
{% endhighlight %}


# Bind Openwhisk to Cloudant
Once the new user is created follow [these instructions](https://new-console.ng.bluemix.net/docs/openwhisk/openwhisk_catalog.html#openwhisk_catalog_cloudant_outside) to bind
the username and password to the new user that was created.  Replace 'MYUSERNAME', 'MYPASSWORD' with the key and password for the new user that was created.  Replace 'MYCLOUDANTACCOUNT.cloudant.com' with the Cloudant hostname you provided by Bluemix.

{% highlight shell %}
./wsk package bind /whisk.system/cloudant blog_news -p username 'MYUSERNAME' -p password 'MYPASSWORD' -p host 'MYCLOUDANTACCOUNT.cloudant.com'
{% endhighlight %}

Retrieve the name of the package binding and note it.
{% highlight shell %}
./wsk package list

packages
/user@us.ibm.com_blog/blog_news                                         private
{% endhighlight %}


# Post a sample news item from the command line

{% highlight shell %}
./wsk action invoke /user@us.ibm.com_blog/blog_news/write --blocking --result --param dbname blog_news --param doc '{"title":"news 3", "link":"http://www.example.com/news3", "excerpt":"Example", "timestamp":"1471314119"}
{% endhighlight %}
Result
{% highlight javascript %}
{
    "id": "5fb45105ac585df3eb5b4746ccc54120",
    "ok": true,
    "rev": "1-a0ac96533ae7991b23ea2f4d76d48b6b"
}
{% endhighlight %}

After executing that command the sample news item should show up in the blog's news article list.
{% include gallery id="whisk_inserted_news" caption="News items inserted into Cloudant by Openwhisk"%}

You can now use the command line Openwhisk to push news articles directly to your blog without having to edit database documents directly on Cloudant.


# Resources
Should you run into any issues consult the [BlueMix Documentation](https://console.ng.bluemix.net/docs/), [Openwhisk Documentation](https://new-console.ng.bluemix.net/docs/openwhisk/index.html), [Cloudant Documentation](https://docs.cloudant.com/) or [reach out to me on twitter](https://twitter.com/boc_tothefuture)
