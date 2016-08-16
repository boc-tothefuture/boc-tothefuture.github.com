---
title:  "Openwhisk Setup"
date:   2016-08-15 17:00:00 -0400
categories: blog cloudant bluemix Openwhisk
excerpt: "Setup Openwhisk"
openwhisk_setup:
  - url: news/openwhisk_setup.jpg
    image_path: news/openwhisk_setup.jpg
    alt: "Openwhisk setup"
    title: "Openwhisk setup instructions"
cloudant_reader_writer:
  - url: news/cloudant_reader_writer.jpg
    image_path: news/cloudant_reader_writer.jpg
    alt: "Cloudant read and write user"
    title: "Cloudant read and write user"
whisk_inserted_news:
  - url: news/whisk_inserted_news.jpg
    image_path: news/whisk_inserted_news.jpg
    alt: "News documents created by whisk"
    title: "News documents created by whisk"
---
<br>
![News](/images/news/whisk.jpg)
<br>
<br>

# Openwhisk

This is the third article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and Cloudant on the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section for a blog. This post will explore how to setup Openwhisk, which will be configured to automatically update the news page.

## Previous Articles

1. [Setting up Cloudant]({% post_url 2016-08-10-news-setting-up-cloudant %}).
2. [Dynamic GitHub pages]({% post_url 2016-08-10-news-setting-up-cloudant %}).

<br>

# Openwhisk Setup

To setup Openwhisk go to [this page](https://new-console.ng.bluemix.net/openwhisk/cli) on Bluemix.  The documentation on that page will guide you through a 3 step process to for Openwhisk setup.
{% include gallery id="openwhisk_setup" caption="Openwhisk setup instruction" %}

Execute the supplied test command and verify you get the expected output.

{% highlight shell %}
 ./wsk action invoke /whisk.system/samples/echo -p message hello --blocking --result
{% endhighlight %}

{% highlight javascript %}
{
    "message": "hello"
}
{% endhighlight %}

# Update Cloudant
In a [previous post]({% post_url 2016-08-10-news-setting-up-cloudant %}) we manually updated news articles by making Cloudant documents on the web interface.  In this post we are going to configure Openwhisk to push updates into Cloudant.

## Create a user that can read and write
Follow the steps in the [previous post]({% post_url 2016-08-10-news-setting-up-cloudant %}) for creating a new user.  When creating this new user enable both \_reader and \_writer permissions.

Store the password for this user somewhere - you can't retrieve it later.
{: .notice--warning}

{% include gallery id="cloudant_reader_writer" caption="Cloudant user with read and write" %}



# Bind Openwhisk to Cloudant
Once the new user is created follow [this instructions](https://new-console.ng.bluemix.net/docs/openwhisk/openwhisk_catalog.html#openwhisk_catalog_cloudant_outside) to bind
the username and password to the new user that was created.  Replace the 'MYUSERNAME', 'MYPASSWORD' with the key, password for the new user that was created.  Replace 'MYCLOUDANTACCOUNT.cloudant.com' with the cloudant hostname you created in the previous posts.

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


# Resources
Should you run into any issues consult the [BlueMix Documentation](https://console.ng.bluemix.net/docs/), [Openwhisk Documentation](https://new-console.ng.bluemix.net/docs/openwhisk/index.html), [Cloudant Documentation](https://docs.cloudant.com/) or [reach out to me on twitter](https://twitter.com/boc_tothefuture)
