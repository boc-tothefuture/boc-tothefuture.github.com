---
title:  "Services, not scripts"
date:   2016-08-02 12:58:00 -0400
categories: devops services chef
excerpt: "Use native operating system service mechanisms."
---
<br>
![Services](/images/gears.jpg)
<br>
<br>
One of the first decisions that need to be made as you move to an automated infrastructure is how to handle services and service like items.

# Where did we come from?

Prior to our conversion to a fully automated platform, we made heavy use of scripts to manage service state.  Most services, such as web servers and application servers
would get a control script placed down as part of the install.  The script would be responsible for starting, stopping and testing items that we managed like services.
As we started our transition it became clear that the ad-hoc script method didn't scale under heavy automation.  Drawbacks include:

 * There was no centralized understanding of service state
 * A different place to start and stop each service with different flags
 * Lots of duplicate script code.

# Change friction

While we were converting one of our products to work in the new environment, we openly debated if we should make this change.  The question was asked during the code review:

> “I am trying to understand what this new approach buys me? It worked fine the other way and at this point better and without all these hassles.”

# Why native services?

After some vigorous debate, we decided we had multiple reasons for pushing forward using native services:

* It is fairly standard across the industry, especially the open source industry from where we started to consume more and more products.
* We can easily select services to start automatically at boot where appropriate, reducing the need for someone to log in or a chef converge to run to bring a node into the expected state.
* You can log into a node, look at the list of services and know what should or should not be running on that node.
* Stopping/Starting/Restarting is uniform across all products/services.  No locations, or special flags to learn, etc.
* It is very easy to automate service stop/start across both [UrbanCode Deploy](http://www.ibm.com/software/products/en/ucdep)(UCD) and [Chef](http://www.chef.io) when everything is a service.  Chef will ensure the service is there, UCD will know it can stop/start the appropriate service.
* When things are a service, we can have Chef shut them down during maintenance and bring them back when maintenance is over.


# What were some things we learned along the way?

* You need to automate when to start and stop services in Chef consistently across services.  
* If a service doesn't background itself [just use](http://jtimberman.housepub.org/blog/2012/12/29/process-supervision-solved-problem/) [runit](https://github.com/chef-cookbooks/runit).  You will save yourself a lot of repeated code and edge cases.
* Validate that all systems that do background have the appropriate scripts placed in this services mechanism for whatever platform you are deploying on.  This consistency is key for high level orchestration.
