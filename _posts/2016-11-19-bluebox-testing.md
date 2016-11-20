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
About two years ago the I started leading the conversion from home grown configuration management and automation to [Chef](https://www.chef.io/).  This was part of an initiative to move from a
traditional infrastructure to a Platform as a Service(PaaS) built on cloud technologies. Part of this transistion included moving from ad-hoc to more formal methods of testing including static analysis, unit and integration testing.  

Formal testing provided many benefits:

* Static analysis creates a uniform code base
* Static analysis enforces both community and team best practices
* Integration and unit testing catch errors before deployment

Formal testing in our environment has one major downside echoed by many many people on our team

> This is SOOOOO SLOW.  Why do I want to do this when I just SSH to a node make a change and be done?  

Despite the benefits of testing, the speed issue hurts adoption of a structured infrastructure management approach.  

# Testing impediments
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

* Hands off - openstack troubleshooting, upgrades, security fixes performed by the Bluebox team
* Rapid provisioning times
* Good end user support model  

# Testing environment
[Test Kitchen](http://kitchen.ci/) coupled with [serverspec](http://serverspec.org/) is the testing harness used for both unit testing and integration testing.  Test kitchen can be used to test locally using [Vagrant](https://www.vagrantup.com/) to manage VMWare or VirtualBox instances.  Test Kitchen can also be used to remotely against Openstack, AWS, or other cloud providers.

## Unit tests
Servespec is used to write unit tests against internally developed cookbooks.  These unit tests are executed by the test kitchen testing harness.  In general these tests are restricted to
only test that the cookbook is correctly functioning and to prevent regressions.  This type of test is usually faster than the integration tests.

## Integration tests
Servespec is used to write integration tests against internally developed role cookbooks. In the platform all depoyments use [role cookbooks](http://realityforge.org/code/2012/11/19/role-cookbooks-and-wrapper-cookbooks.html).  These role cookbooks specify and invoke all dependencies required to fully construct a node
from scratch.  Essentially, role cookbooks tie together other cookbooks and specify exact dependencies to ensure repeatability of deploys.

Why use role cookbooks?

* Reduced failures during provisioning compared to using individual cookbooks and standard chef roles
* The user can test that everything fits together before pushing it out into production

In general, most cookbooks can be tested quickly because they represent just a fraction of the functionality of a node.  However, role cookbooks can take quite a bit longer.  

# Starting point

Just switching to remote testing with no optimizations was not the panacea I hoped for:

## Cookbook (Unit) test
{% highlight console %}
kitchen converge all -c
<Output removed for brevisity>
-----> Kitchen is finished. (5m54.30s)
{% endhighlight %}


## Role cookbook (Integration) test
{% highlight console %}
kitchen converge all -c
<Output removed for brevity>
-----> Kitchen is finished. (18m9.95s)
{% endhighlight %}

18 minutes and 9 seconds..  
Work is going to be necessary to get this number into an acceptable range

# Stay tuned
The next post will cover how to tackle optimization problems for remote testing

# Questions?
[Reach out to me on twitter](https://twitter.com/boc_tothefuture)
