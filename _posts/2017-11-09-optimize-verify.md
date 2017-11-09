---
title:  "Optimize kitchen verify with Chef"
date:   2017-11-09 12:00:00 -0400
categories: openstack chef
excerpt: "Optimize kitchen verify with remote testing"
---

<br>
![Bluebox](/images/bluebox/verify.jpg)
<br>
<br>

# Optimizing remote testing
The [last article]({% post_url 2017-08-24-optimize-converge %}) described how to optimize the converge phase of kitchen testing. This post will describe methods to optimize the kitchen verify step.

The current duration for each stage.

| Stage         | Duration (seconds)     | % Total  |
| ------------- | ------------- | ----- |
| Create        | 20 seconds  | 1% |
| Prepare       | 3.4 seconds     |   1% |
| Converge      | 10 minutes 14 seconds       |   95% |
| Verify        | 11 seconds      |    2% |



<br>
# Previous Articles
1. [Remote testing in Bluebox with Chef]({% post_url 2016-11-19-bluebox-testing %})
2. [Optimization overview of Bluebox with Chef]({% post_url 2016-11-28-bluebox-optimizing %})
3. [Optimizing boot]({% post_url 2016-11-29-bluebox-optimize-boot %})
4. [Optimizing create]({% post_url 2016-12-08-bluebox-optimize-create %})
5. [Optimizing prepare]({% post_url 2017-08-06-optimize-prepare %})
5. [Optimizing converge]({% post_url 2017-08-24-optimize-converge %})

<br>
# Kitchen Verify
The verify stage of test kitchen executes one or more tests against the converged test instance. This stage should, in most instances, be the quickest. Verify really has two distinct segments:
*  Prepare for verification
*  Run verification

## Prepare
This stage ensures all the appropriate code is in place to perform the system verification.  This can include installing the required gems and uploading the tests. This stage may be slightly optimized.

## Verification
This is the execution of the test code.  The length of this stage depends on the type of tests being executed and how quickly they run.  Because this can change from cookbook to cookbook, there are no general optimization strategies.

# Prepare optimization
There are a number of steps that test kitchen executes during the prepare stage for verification.  Primarily, test kitchen is installing and configuring a set of gems to assist with verification.  The steps include to prepare include:

1. Installing the [busser gem](https://github.com/test-kitchen/busser).
2. Installing the [serverspec gem](http://serverspec.org/).
3. Setup busser
4. install the busser server-spec plugin

Doing these steps on average takes about 5 seconds.  We can reach into our bag of tricks from our [previous article]({% post_url 2017-08-24-optimize-converge %}) on kitchen converge to optimize this stage.


## Preform prepare optimization at boot.
 The method below optimizes kitchen converge by taking advantage of the fact that the system is running while test-kitchen is uploading cookbooks in the prepare and converge stages. This can be performed using the `at` command inside of cloud-init.  This method results the command immediately exiting and running the job in the background. If `at` was not used, cloud-init would delay the boot process until the YUM cache was created, negating any potential gains.

The cloud init configuration looks like this:

{% highlight bash %}
- path: /tmp/install_busser
  owner: root:root
  permissions: '0644'
  content: |
    # Pre install the kitchen busser command
    export BUSSER_ROOT="/tmp/verifier/"
    export GEM_HOME="/tmp/verifier/gems"
    export GEM_PATH="/tmp/verifier/gems"
    export GEM_CACHE="/tmp/verifier/gems/cache"
    su vagrant -c '/opt/chef/embedded/bin/gem install busser --no-rdoc --no-ri'
    su vagrant -c '/opt/chef/embedded/bin/gem install serverspec --no-rdoc --no-ri'
    su vagrant -c '/tmp/verifier/gems/bin/busser setup'
    su vagrant -c '/tmp/verifier/gems/bin/busser plugin install busser-serverspec‘

runcmd:
- 'at now -M -f /tmp/install_busser'

{% endhighlight %}

This update to the cloud-init file saves about 5 seconds for every converge.


# Summary

Package optimization and yum-cache enhancements provide a measurable performance improvement in the converge phase:

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Kitchen Verify  | 11 seconds | 2% | 6 seconds | 54.55% | 1%

5 seconds is removed from every verify by using the cloud-init optimization for Busser creation.

Current stage times after converge optimization:

| Stage         | Duration (seconds)     | % Total  |
| ------------- | ------------- | ----- |
| Create        | 20 seconds  | 1% |
| Prepare       | 3.4 seconds     |   1% |
| Converge      | 6 minutes 59 seconds       |   92% |
| Verify        | 6 seconds      |    1% |

That is it!  I hope you enjoyed this series on optimizing remote testing using Chef.

# Questions?
[Reach out to me on twitter](https://twitter.com/boc_tothefuture)
