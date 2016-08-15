---
title:  "Zero to Cloudant in 5 minutes"
date:   2016-08-10 20:20:00 -0400
categories: blog cloudant bluemix
excerpt: "Setting up Cloudant in BlueMix"
create_user:
  - url: news/create_user.jpg
    image_path: news/create_user.jpg
    alt: "Create a Read-Only User"
    title: "Create a read only user in Cloudant"
data_services:
  - url: news/bluemix_data_services.jpg
    image_path: news/bluemix_data_services.jpg
    alt: "BlueMix Data Services"
    title: "BlueMix Data Services"
select_plan:
  - url: news/cloudant_plan.jpg
    image_path: news/cloudant_plan.jpg
    alt: "Select a Cloudant plan"
    title: "Select a Cloudant plan"
document_url:
  - url: news/document_url.jpg
    image_path: news/document_url.jpg
    alt: "Determine document URL"
    title: "Determine document URL"
---
<br>
![News](/images/news/news.jpg)
<br>
<br>

# Creating an automatically updating news page

This is the first article in a series on how to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating news section.

# Persisting Data

![News](/images/news/cloudant.jpg)

I will be consuming the [Cloudant](http://www.ibm.com/Cloudant) [Database as a Service](https://en.wikipedia.org/wiki/Cloud_database) to persist the articles I want for my news section of this blog.  Cloudant DBaaS has several features that make it a good fit, including:

 * A JSON document database is a natural fit for storing news article information.
 * HTTP based API to update and access database
 * DBaaS - no servers to maintain, quick and easy setup

 <br>

# Getting started Cloudant on BlueMix

1. Create a BlueMix account
2. Find the Cloudant service
3. Select a Cloudant plan
4. Add Cloudant to your BlueMix

## Create your BlueMix account
You will need to register to get started with BlueMix.  Click [here](https://console.ng.bluemix.net/registration/) to create a BlueMix account.

## Find the Cloudant Service
* Select [catalog](https://console.ng.bluemix.net/catalog/) from the top navigation bar
* Check Data and Analytics on the filter on the left navigation bar.  

You should see a screen like this
{% include gallery id="data_services" caption="BlueMix Data Services" %}

## Select Cloudant NoSQL DB Service

![Cloudant Service](/images/news/cloudant_service.jpg)

## Select Cloudant Plan
From here, you can add and configure the Cloudant service.  I left all the default values and clicked create.
{% include gallery id="select_plan" caption="Select a Cloudant Plan" %}

## Add Cloudant to your BlueMix
BlueMix will configure your Cloudant service.  Once it is done click the Launch icon to open the Cloudant console.

![Cloudant Launch](/images/news/cloudant_launch.jpg)

Refer to the [BlueMix docs](https://console.ng.bluemix.net/docs/) for more details or troubleshooting.

# Setup Cloudant

Now that the Cloudant service is added, it needs to be configured for our use case.
<br>

Steps include:

1. Create a DB
2. Add a read only user
3. Put our first news item in the database.

## Create a Cloudant Database
Click on the upper right hand corner link that says create database.  I named mine blog_news.

![Create Database](/images/news/create_database.jpg)

## Add a read only user
To access this from our the web browser I want to create a read only user ID.
<br>

To make and configure the user:

* Select permissions from the left nav
* Click Generate API key
* Validate the new API key only has reader permissions.

When done your screen should look something like this
{% include gallery id="create_user" caption="Create a read only user in Cloudant" %}

Store the password for this user somewhere - you can't retrieve it later.
{: .notice--warning}

## Create a test news item

To create a test news item:

 * Navigate back to the All Documents section on the left Nav
 * Click the plus sign next to all documents
 * Select New Doc

Your screen should look like this right before you click 'New Doc'
![New Document](/images/news/create_document.jpg)

This should open the Cloudant online JSON text editor. Let's author our first news item document. Add the following fields for all news article documents.
In addition to the \_id field inserted automatically by Cloudant we will

{% highlight javascript %}
{
  "title": "News Article",
  "link": "www.example.com/interesting_article.html",
  "excerpt": "Summary of this article",
  "timestamp": "1470844114"
}
{% endhighlight %}

Your screen should look something like this.
![New Document](/images/news/example_document.jpg)

Record the automatic document id assigned by Cloudant we will need that in the next section.
{: .notice--warning}

Click the create document button to insert the document into Cloudant.

# Test Cloudant
To test our process we will retrieve the document we just inserted into Cloudant.

1. Get the URL for the document
2. Craft a curl to retrieve the document
3. Validate document exists and matches what we submitted.

## Get the URL for the document
In the all documents view, click on the pencil icon next to the document whose ID matches the document ID you noticed in the previous section.
This should bring up an editable version of the document you crafted earlier.  In the upper right hand side of the browser click the API icon and then
select copy URL.  

Your screen should look like this
{% include gallery id="document_url" caption="Determine the document URL" %}

Create a curl like the one below replacing <key> <password> below with the key and password Cloudant generated for you earlier. The <url> should be the one you just
copied from the web interface.

## Craft your curl
{% highlight shell %}
curl -u <key>:<password> <url>
{% endhighlight %}

This is what it looks like with some fake values in place.
{% highlight shell %}
curl -u icheteegesomenewh:ec4b7670b96ef1359c2d8342a6c https://2619ca16-f337-72b9-b353-00009fcf33d7-bluemix.cloudant.com/blog_news/cfaa26fca49a1734f3a25cbc7fecfb02
{% endhighlight %}

## Compare documents
The results of the curl, should be the same JSON document you just finished creating with a revision id automatically inserted by Cloudant.
{% highlight javascript %}
{ "_id":"aacf26fac49a1734f3a25cbc7fecfb02",
  "_rev":"1-84896a97574f81ad9969b6c773d42a30",
  "title":"News Article",
  "link":"www.example.com/interesting_article.html",
  "excerpt":"Summary of this article",
  "timestamp":"1470844114"}
{% endhighlight %}


# Resources
Should you run into any issues consult the [BlueMix Documentation](https://console.ng.bluemix.net/docs/), the [Cloudant Documentation](https://docs.cloudant.com/) or [reach out to me on twitter](https://twitter.com/boc_tothefuture)
