---
title:  "Optimize kitchen prepare with Chef"
date:   2017-08-06 12:00:00 -0400
categories: openstack chef
excerpt: "Optimize kitchen prepare with remote testing"
---

<br>
![Bluebox](/images/bluebox/prepare.jpg)
<br>
<br>

# Optimizing remote testing
A [previous article]({% post_url 2016-12-08-bluebox-optimize-create %}) described how to optimize the image create part of the kitchen create stage. This post will focus on the kitchen prepare step.

The current duration for each stage.

| Stage         | Duration (seconds)     | % Total  |
| ------------- | ------------- | ----- |
| Create        | 20 seconds  | 1% |
| Prepare       | 7 minutes 36 seconds     |   41% |
| Converge      | 10 minutes 14 seconds       |   67% |
| Verify        | 11 seconds      |    1% |


<br>
# Previous Articles
1. [Remote testing in Bluebox with Chef]({% post_url 2016-11-19-bluebox-testing %})
2. [Optimization overview of Bluebox with Chef]({% post_url 2016-11-28-bluebox-optimizing %})
3. [Optimizing boot]({% post_url 2016-11-29-bluebox-optimize-boot %})
4. [Optimizing create]({% post_url 2016-12-08-bluebox-optimize-create %})

<br>
# Kitchen Prepare
The prepare stage of test kitchen executes a few required tasks before the converge step.  Those tasks include:
*  Installing chef client test server (if not installed or at the right level)
*  Starting the chef zero server on the test server
*  Uploading cookbooks

These steps all seem fairly simple, what is taking up seven and a half minutes?

## Installing Chef
 The .kitchen.yml, can specify via the provisioner directive a version of chef to verify is installed or install if missing or not at the right level.  This is done using the product_name and product_version directives.  An example configuration to verify chef 13.0.118 is installed would look like this:
{% highlight yaml %}
 provisioner:
   name: chef_zero
   product_name: chef
   product_version: 13.0.118
{% endhighlight %}

To optimize testing speed, the image being tested against already has Chef installed at the correct version, which is the same version that is running in production. The test kitchen file being used for these tests do not specify a product_name or product_version so this step is skipped. This step isn't contributing to the lengthy prepare time and there is nothing to further optimize.

## Start Chef-Zero
What is Chef Zero?  According to the [readme](https://github.com/chef/chef-zero/blob/master/README.md)
> Chef Zero is a simple, easy-install, in-memory Chef server that can be useful for Chef Client testing and chef-solo-like tasks that require a full Chef Server. <br>
... <br>
Because Chef Zero runs in memory, it's super fast and lightweight. This makes it perfect for testing against a "real" Chef Server without mocking the entire Internet.

This should be quick to start and shouldn't be significantly contributing to the prepare time. Nothing to optimize in this step.

## Uploading cookbooks
The process of elimination has left us with only one likely culprit. Inspecting the logs confirms the suspicion.

{% highlight shell %}
Transferring files to <cloud-cmusta-prd-RHEL7>
D TIMING: scp async upload (Kitchen::Transport::Ssh)
D TIMING: scp async upload (Kitchen::Transport::Ssh) took (7m36.16s)
{% endhighlight %}

The real question is why is it taking so long and why wasn't it an issue when testing locally? <br>

The answer is a combination of the number of files being uploaded and the method used by test kitchen to upload the files. Some of the role cookbooks have over 1,000 files and test kitchen uploads those one a time.  When the system is local, that is quick. However, with remote testing every one of those files transferred requires a lot of communication that is delayed because of the distance between the user and the remote testing location.  
<br>
What are the options to resolve this?

Test kitchen supports plugins and developers have contributed a wide array of plugins to enhance test kitchen.

## speedy-ssh
One of those plugins is called [speedy-ssh](https://github.com/criteo/kitchen-transport-speedy)
> This gem is transport plugin for kitchen. It signicantly improves file synchronization between workstation and boxes using archive transfer instead of individual transfers.
The transport only works where ssh works and requires tar to be present both on the workstation and in the box.

This gem takes all cookbooks, creates a tar file, uploads it, connects to the remote machine and untars the file. Then it repeats the same process for roles, data bags, etc.  This should take the number of unique file transfers from the local machine to the remote test machine from over a 1,000 to a few.  
{% highlight shell %}
D  /tmp/cloud-cmusta-prd-RHEL7-sandbox-20170507-15058-185i7ay/cookbooks contains 1146
D  Calling regular upload for /tmp/cloud-cmusta-prd-RHEL7-sandbox-20170507-15058-185i7ay/cookbooks.tar to /tmp/kitchen
D  TIMING: scp async upload (Kitchen::Transport::Ssh)
D  TIMING: scp async upload (Kitchen::Transport::Ssh) took (0m1.32s)
{% endhighlight %}
Cookbooks now take 1.32 seconds, a huge improvement.  The total process takes around 5.6 seconds as it is repeated for the different data types that are uploaded to remote test server.  5.6s would be an acceptable improvement, but there are a few other plugins worth testing.

## kitchen-sync
The next plugin evaluated is [kitchen-sync](https://github.com/coderanger/kitchen-sync). This plugin supports two modes, sftp and rsync. The fastest mode is rsync.
>This is the fastest mode, but it does have a few downsides. The biggest is that you must be using ssh-agent and have an identity loaded for it to use. It also requires that rsync be available on the remote side.

The setup being tested has ssh-agent and an identity file loaded, making this a viable option.
{% highlight shell %}
 [rsync] Time taken to upload /tmp/cloud-cmusta-prd-RHEL7-sandbox-20170507-13159-163maoo/cookbooks;/tmp/cloud-cmusta-prd-RHEL7-sandbox-20170507-13159-163maoo/nodes;/tmp/cloud-cmusta-prd-RHEL7-sandbox-20170507-13159-163maoo/data_bags to vagrant@10.0.0.136<{:user_known_hosts_file=>"/dev/null", :paranoid=>false, :port=>22, :compression=>true, :compression_level=>9, :keepalive=>true, :keepalive_interval=>60, :timeout=>15, :keys_only=>true, :keys=>["/home/path/key/key_rsa"], :auth_methods=>["publickey"], :user=>"vagrant"}>:/tmp/kitchen: 2.52 sec
D [rsync] Using fallback to upload /tmp/cloud-cmusta-prd-RHEL7-sandbox-20170507-13159-163maoo/cache;/tmp/cloud-cmusta-prd-RHEL7-sandbox-20170507-13159-163maoo/dna.json;/tmp/cloud-cmusta-prd-RHEL7-sandbox-20170507-13159-163maoo/client.rb;/tmp/cloud-cmusta-prd-RHEL7-sandbox-20170507-13159-163maoo/validation.pem
D TIMING: scp async upload (Kitchen::Transport::Ssh)
D TIMING: scp async upload (Kitchen::Transport::Ssh) took (0m0.91s)
{% endhighlight %}
Less than a second for the measured transport of over 1,000 files and the total time for this step is now decreased to 3.4 seconds. This process appears to be quicker than speedy-ssh because only a single connection and transfer is performed that includes all data types (cookbooks, roles, environments) needed for testing. A winner has emerged. If you are doing remote testing, then kitchen-sync can highly optimize your test process.

# Summary

The kitchen-sync plugin provides a huge performance improvement in the prepare phase:

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Kitchen Prepare  | 7m36s | 41% | 3.4s | 99.25% | 1%

With prepare optimization complete, the total time before converge starts is down to 25 seconds and the prepare stage is now just 1% of the total operation. The moral of the story here is that it is important to minimize the number of round-trips required when using remote testing.

| Stage         | Duration (seconds)     | % Total  |
| ------------- | ------------- | ----- |
| Create        | 20 seconds  | 1% |
| Prepare       | 3.4 seconds     |   1% |
| Converge      | 10 minutes 14 seconds       |   95% |
| Verify        | 11 seconds      |    2% |

Next up for optimization? Converge.

# Questions?
[Reach out to me on twitter](https://twitter.com/boc_tothefuture)
