---
title:  "A unique flow in Openwhisk"
date:   2016-08-17 10:00:00 -0400
categories: blog cloudant bluemix Openwhisk
excerpt: "Filtering unique news posts for a blog with Openwhisk"
duplicate_posts:
  - url: news/duplicate_posts.jpg
    image_path: images/news/duplicate_posts.jpg
    alt: "Duplicate news posts"
    title: "Duplicate news posts"
cloudant_index:
  - url: news/cloudant_index.jpg
    image_path: images/news/cloudant_index.jpg
    alt: "Create a Cloudant index"
    title: "Create a Cloudant index"
cloudant_create_index:
  - url: news/cloudant_create_link_index.jpg
    image_path: images/news/cloudant_create_link_index.jpg
    alt: "Create the link index"
    title: "Create a link index"  
no_duplicate_posts:
  - url: news/no_duplicate_posts.jpg
    image_path: images/news/no_duplicate_posts.jpg
    alt: "No duplicate news posts"
    title: "No duplicate news posts"
---
<br>
![News](/images/news/unique.jpg)
<br>
<br>

# Ensuring unique news articles

This is the forth article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and Cloudant on the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section for a blog. This post will describe how to configure your Openwhisk chain to make sure news items on your blog are only posted once.

## Previous Articles

1. [Setting up Cloudant]({% post_url 2016-08-10-news-setting-up-cloudant %})
2. [Dynamic GitHub pages]({% post_url 2016-08-10-news-setting-up-cloudant %})
3. [Mixing in Openwhisk]({% post_url 2016-08-16-news-openwhisk-setup %})

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


Both requests return OK and duplicate entries exist on the news page.

{% include gallery id="duplicate_posts" caption="News page with duplicate posts" %}


## Steps to ensure unique articles

1. Create a Cloudant index on the link field
2. Filter news articles with links that already exist in Cloudant
3. Insert filtered news articles

<br>

# Another Cloudant index

Creating the initial Cloudant index was described in detail in [this post]({% post_url 2016-08-10-news-setting-up-cloudant %}).  To create another index open up the Cloudant portal, select Databases from the left navigation section and then select your blog news database. Select the plus sign next to Design Documents then select query indexes.
{% include gallery id="cloudant_index" caption="Create a new query index" %}

In the next screen you can create your index.  Unlike the previous index, this time we want to index the link of the news articles so we can make sure all news artciles are unique. From the supplied default index example in cloudant replace 'foo' with 'link' and click create index.
{% include gallery id="cloudant_create_index" caption="Link index creation" %}

With the index created, Cloudant can now be quickly queried to see if a link already exists.

# Filter news articles
A custom Openwhisk action will be used to filter news articles to ensure uniqueness.  This filter will be written in Javascript.

## Create a file called insert_article.js
{% highlight shell %}
vi insert_article.js
{% endhighlight %}

The content of the insert_article action in its entirety
{% highlight javascript %}
function main(params) {

  whisk.invoke({
    name: '/user@us.ibm.com_blog/blog_news/exec-query-find',
    parameters:{
      dbname: 'blog_news',
      query:  {
        "selector": {
          "link": {
            "$eq": params.link
          }
        },
        "fields": [
          "_id"
        ],
        "sort": [
          {
            "link": "desc"
          }
        ]
      }
    },
    blocking: true,
    next:  function(error, activation) {
      console.log('error:', error, 'activation:', activation);
      if(error){
        whisk.error(error);
      }
      else{
        if( activation.result.docs.length > 0 ) {
          whisk.done({msg:'Link already present. Ignoring article.'});
        }
        else{
          whisk.invoke({
            name: '/user@us.ibm.com_blog/blog_news/write',
            parameters:{
              dbname: 'blog_news',
              doc: {
                "title": params.title,
                "link": params.link,
                "excerpt": params.excerpt,
                "timestamp": String(params.timestamp)
              }
            },
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
          return whisk.async();
        }
      }
    }
  });
  return whisk.async();
}
{% endhighlight %}

The nesting of the javascript makes this action look complicated, but the logic is fairly simple.

1. Action parameters are passed in with the params object.  This action uses title, link, excerpt and timestamp.
2. Use an asynchronous whisk action to query Cloudant for any documents containing the link passed in as a parameter.
3. If the results set contains 1 or more documents, end the Openwhisk chain with a call to whisk.done.  Otherwise invoke another asynchronous Cloudant whisk action to insert the article.
4. Execute a whisk.done call after adding the article to Cloudant.


## Create and upload the action to Openwhisk
{% highlight shell %}
./wsk action create insert_article insert_article.js
ok: created action insert_article
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
