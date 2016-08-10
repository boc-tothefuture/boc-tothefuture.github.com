---
title:  "Configuring Cloudant"
date:   2016-08-09 21:30:00 -0400
categories: blog cloudant bluemix
excerpt: "Setting up Cloudant in BlueMix"
---
<br>
![News](/images/news/news.jpg)
<br>
<br>

# Creating an automatically updating news page

This is the first article in a series on to use [GitHub pages](https://pages.github.com/) (the service that hosts this blog) and the [BlueMix PaaS](http://www.ibm.com/BlueMix) to create an automatically updating
news section for a blog.

# Persisting Data
![News](/images/news/cloudant.jpg)

I will be consuming the [Cloudant](http://www.ibm.com/Cloudant) [Database as a Service](https://en.wikipedia.org/wiki/Cloud_database) to perist the articles I want to show up in the news section
of this blog.  Cloudant DBaaS is a good fit for this use case for several reasons.

 * A JSON document style database is a natural fit for storing news article information.
 * HTTP based API to update and access database
 * DBaaS - no servers to maintain, quick and easy setup

 <br>
 <br>

# Getting started Cloudant on BlueMix

* Create a BlueMix account
* Find the Cloudant Service
* Select a Cloudant Plan
* Add Cloudant to your BlueMix

## Create your BlueMix account
You will need to register to get started with BlueMix.  [create a bluemix account.](https://console.ng.bluemix.net/registration/).

## Find the Cloudant Service
* Select [catalog](https://console.ng.bluemix.net/catalog/) from the top navigation bar
* Check Data and Analytics on the filter on the left navigation bar.  

You should see a screen like this

![BlueMix Data Services](/images/news/bluemix_data_services.jpg)

## Select Cloudant NoSQL DB Service

![Cloudant Service](/images/news/cloudant_service.jpg)

## Select Cloudant Plan
From here, you can add the Cloudant service to your account.  You can leave the default values and click create.

![Cloudant Plan](/images/news/cloudant_plan.jpg)

## Add Cloudant to your BlueMix
BlueMix will configure your Cloudant service.  Once it is done click the Launch icon to open the Cloudant console.

![Cloudant Launch](/images/news/cloudant_launch.jpg)

Should you get lost following my directions the [docs](https://console.ng.bluemix.net/docs/) for BlueMix are some of the best.  

# Setup Cloudant

Now that the Cloudant service is added we will need to follow some steps to configure it for our purposes.

* Create a DB
* Add a read only user
* Put our first news item in the database.

# Test Cloudant
