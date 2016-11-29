---
title:  "Optimization overview of Bluebox with Chef"
date:   2016-11-28 12:00:00 -0400
categories: openstack chef Bluebox
excerpt: "Systematic approach to optimizing chef testing using Bluebox"
---

<br>
![Bluebox](/images/bluebox/fast.jpg)
<br>
<br>

# Optimizing remote testing
A [previous article]({% post_url 2016-11-19-bluebox-testing %}) described all the reasons to test remotely using Bluebox.  This post is going to focus on how to create a systematic approach to optimizing the test times for remote testing in bluebox.  

# Previous Articles

1. [Remote testing in Bluebox with Chef]({% post_url 2016-11-19-bluebox-testing %})



# Set goals
When forming an optimization plan it is important to have an end goal.  Determine, the acceptable time for this operation to complete.  The goal time will impact the major focus areas for optimization and how the optimization is approached.  

After discussion with a major stakeholder of our project, the following non-functional requirement for performance was selected.

> I need to test across all sponsorship event websites across all suites in 5 minutes.

There are 6 sponsorship websites, with 3 suites (test, pre-production and production) each.  Therefore, the system must support running 18 tests in under 5 minutes without compromising test integrity.

From the previous blog post, the time with no optimization for a single test was 18 minutes and 8 seconds.
{% highlight console %}
kitchen converge all -c
<Output removed for brevity>
-----> Kitchen is finished. (18m9.95s)
{% endhighlight %}

Taking 18 minutes 9 seconds to 5 minutes, is a 72% reduction in test time. This is a very aggressive target, will require some thorough optimzation.

Bluebox/Openstack can run tests concurrently, however I anticipate hitting some scaling issues with the tooling when attempting to run 18 concurrently.

# Optimization strategy
With a goal set, the next step in optimization is to break the problem down into different chunks that can be independenatly optimized.

There are 4 main stages to optimize during a kitchen test

* Create - Creation and startup of the image to test against
* Prepare - The setup duration between create and start of the converge
* Converge - This is the cookbook converge time
* Verify - This is how long it takes to run the tests

Because the optimization goal is so aggressive, iteration of improvements over these stages may be necessary.  Tackling the lowest hanging furit from each stage until the goal is met or it is determined that no further optimizations may occur without impacting test validity.

# Inital measurements
Before starting an optimization journey, it is important to measure the starting point for each area to keep track of improvements.

## Create
{% highlight console %}
kbb create cmusta-pre-RHEL7
-----> Starting Kitchen (v1.11.1)
-----> Creating <cmusta-pre-RHEL7>...
       OpenStack instance with ID of <0dc3936d-cb74-4743-a1a0-e756b0bf4e35> is ready.
       Attaching floating IP from <external> pool
       Attaching floating IP <169.50.195.132>
       Waiting for server to be ready...
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       Waiting for SSH service on 169.50.195.132:22, retrying in 3 seconds
       [SSH] Established
       Adding OpenStack hint for ohai
       Finished creating <cmusta-pre-RHEL7> (1m20.93s).
-----> Kitchen is finished. (1m21.57s)
{% endhighlight %}

Creating the image takes 1 minutes and 22 seconds.

## Prepare
Prepare is part of the converge process in test kitchen and is measured as the time between the start of the kitchen converge command and the execution of the first cookbook.  There is not a distinct command for test kitchen, therefore it will be measured by looking at the log.

{% highlight console %}
I, [2016-11-28T22:17:23.348451 #20655]  INFO -- cmusta-pre-RHEL6: -----> Converging <cmusta-pre-RHEL6>...
<... Output trimmed for brevity ...>
I, [2016-11-28T22:24:59.178576 #20655]  INFO -- cmusta-pre-RHEL6: Recipe: chef-sugar::default
{% endhighlight %}

Prepare takes 7 minutes and 36 seconds.

## Converge
The converge is measured by looking in the log and finding the log statement that records the duration of the chef client run.

{% highlight console %}
I, [2016-11-28T22:34:34.315098 #20655]  INFO -- cmusta-pre-RHEL6: Chef Client finished, 467/842 resources updated in 10 minutes 14 seconds
{% endhighlight %}

Converge takes 10 minutes and 14 seconds.


## Verify
{% highlight console %}
kbb verify cmusta-pre-RHEL6
-----> Starting Kitchen (v1.11.1)
-----> Setting up <cmusta-pre-RHEL6>...
       Finished setting up <cmusta-pre-RHEL6> (0m0.00s).
-----> Verifying <cmusta-pre-RHEL6>...
       Preparing files for transfer
-----> Installing Busser (busser)
Fetching: thor-0.19.0.gem (100%)
       Successfully installed thor-0.19.0
Fetching: busser-0.7.1.gem (100%)
       Successfully installed busser-0.7.1
       2 gems installed
       Installing Busser plugins: busser-serverspec
       Plugin serverspec installed (version 0.5.7)
-----> Running postinstall for serverspec plugin
       Suite path directory /tmp/verifier/suites does not exist, skipping.
       Transferring files to <cmusta-pre-RHEL6>
-----> Running serverspec test suite
-----> Installing Serverspec..
Fetching: sfl-2.3.gem (100%)
Fetching: net-telnet-0.1.1.gem (100%)
Fetching: net-ssh-3.2.0.gem (100%)
Fetching: net-scp-1.2.1.gem (100%)
Fetching: specinfra-2.66.0.gem (100%)
Fetching: multi_json-1.12.1.gem (100%)
Fetching: diff-lcs-1.2.5.gem (100%)
Fetching: rspec-support-3.5.0.gem (100%)
Fetching: rspec-expectations-3.5.0.gem (100%)
Fetching: rspec-core-3.5.4.gem (100%)
Fetching: rspec-its-1.2.0.gem (100%)
Fetching: rspec-mocks-3.5.0.gem (100%)
Fetching: rspec-3.5.0.gem (100%)
Fetching: serverspec-2.37.2.gem (100%)
-----> serverspec installed (version 2.37.2)

       Finished in 1.08 seconds (files took 0.48296 seconds to load)
       52 examples, 0 failures

       Finished verifying <cmusta-pre-RHEL6> (0m9.95s).
-----> Kitchen is finished. (0m10.58s)
{% endhighlight %}

Verify takes 11 seconds.

# Summary
This table summarize the test kitchen stages, we will refer to it throughout this series as we optimize our test times.

| Stage         | Duration (seconds)     | % Total  |
| ------------- | ------------- | ----- |
| Create        | 1 minute 22 seconds  | 8% |
| Prepare       | 7 minutes 36 seconds     |   39% |
| Converge      | 10 minutes 14 seconds       |   52% |
| Verify        | 11 seconds      |    1% |

The approach we will take is to optimize the testing process in the order of the kitchen stages.  Next up, will be create optimization.

# Questions?
[Reach out to me on twitter](https://twitter.com/boc_tothefuture)
