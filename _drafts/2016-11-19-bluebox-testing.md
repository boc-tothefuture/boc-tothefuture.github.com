---
title:  "Remote testing in Bluebox with Chef"
date:   2016-11-17 12:00:00 -0400
categories: openstack chef Bluebox
excerpt: "Using Bluebox for Chef testing"

---

<br>
![Bluebox](/images/bluebox/bluebox.jpg)
<br>
<br>

# Infrastructure testing

About two years ago the team started converting from home grown configuration management and automation to [Chef](https://www.chef.io/).  This was part of the initiative to move from a
traditional infrastructure to a Platform as a Service built on cloud technologies.  As part of the transistion to industry standard configuration management and
automation systems another a fundemental shift took palce as testing transitioned from ad-hoc to more formal methods of testing including static analysis, unit and integration testing.  

Formal testing provided many benefits:

* Static analysis helped create a uniform code base
* Static analysis helped enforce both community and team best practices
* Integration and Unit testing helped catch errors before deployment

Formal testing had one major downside echoed by many many people on our team

> This is SOOOOO SLOW.  Why do I want to do this when I just SSH to a node make a change and be done?  

Despite the benefits of testing, the speed issue hurts adoption of a structured infrastructure management approach.  

Why was local laptop based testing so slow?

* Location - This is a world wide team with people working at home and around the globe, sometimes with slow network connections
* Equipment - It is a 4 year hardware refresh cycle, not everyone has the latest Macbook Pro  
* Software - A lot of enterprise software is used, often these are packaged into large artifacts

With a growing global team, it became clear that a new testing methodology was necessary as locally testing on a laptop was becoming untenable.

# Enter Bluebox

I selected [Bluebox](https://www.blueboxcloud.com/) as our remote testing method.  

What is Bluebox?<br>
Bluebox is a managed Infrastructure as a Service offering based on Openstack.

Why Bluebox?

* Openstack troubleshooting, upgrades, security fixes performed by the Bluebox team
* Rapid provisioning times
* Good end user support model  


Need transition here...

# Role cookbooks

In the platform all depoyments use [role cookbooks](http://realityforge.org/code/2012/11/19/role-cookbooks-and-wrapper-cookbooks.html).  These role cookbooks specify and invoke all dependencies required to fully construct a node
from scratch.  Because role cookbooks just tie togehter cookbooks, this means that to deploy a chance into the infrastucture in most circumstances one will first have to make the change to a cookbook and then update the
role cookbook to consume that change.  This means testing of both the cookbook in question and the role cookbook.  

Why use role cookbooks?
* We were experience lots of failures on provision when using individual cookbooks and regular roles which were not tested
* The user gets to test that everything fits together before pushing it out into production

Most cookbooks can be tested quickly because they represent just a fraction of the functionality of a node.  However, role cookbooks can take quite a bit longer.  

# Starting point

Just switching to remote testing with no optimizations was not the panacea I hoped for:

{% highlight console %}
       Finished converging <cmusta-pre-RHEL6> (16m34.67s).
-----> Kitchen is finished. (17m24.34s)
{% endhighlight %}

17 minutes and 24 seconds..  
Some work is going to be necessary to get this number into the acceptable range.

# Next up?
In the next post I will cover the optimizations made to the base image to support rapid remote testing


# Questions?
[Reach out to me on twitter](https://twitter.com/boc_tothefuture)
