---
title:  "Optimize image create in Bluebox with Chef"
date:   2017-01-18 12:00:00 -0400
categories: openstack chef Bluebox
excerpt: "Optimize image create in Bluebox for use with Test Kitchen"
---

<br>
![Bluebox](/images/bluebox/create.jpg)
<br>
<br>

# Optimizing remote testing
A [previous article]({% post_url 2016-11-29-bluebox-optimize-boot %}) described how to optimize the boot part of the create stage. This post will focus on the Bluebox image create step within the kitchen create method.

The current duration for each stage.

| Stage         | Duration (seconds)     | % Total  |
| ------------- | ------------- | ----- |
| Create        | 40 seconds  | 3% |
| Prepare       | 7 minutes 36 seconds     |   39% |
| Converge      | 10 minutes 14 seconds       |   52% |
| Verify        | 11 seconds      |    1% |


# Previous Articles

1. [Remote testing in Bluebox with Chef]({% post_url 2016-11-19-bluebox-testing %})
2. [Optimization overview of Bluebox with Chef]({% post_url 2016-11-28-bluebox-optimizing %})
2. [Optimizing boot]({% post_url 2016-11-29-bluebox-optimize-boot %})



# Image Create
In the last test, image create was measured at 80% of the total kitchen create time.  The time is inflated because the image is not cached on the hypervisors because the image is being replaced every time we optimized OS boot.  To get a more accurate number, create multiple instances of the same image a few times to make sure the image is cached on the hypervisors in Bluebox.  

After a few kitchen creates, the time started to normalize.  In the table below we have replaced the image create original duration with the new normalized time and we will use this as our base time for measure image create optimization.

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Image create  | 17s | 37% | 17s | 0% | 68%
| OS Boot       | 69s| 63% | 8s | 88% | 32%

## Image Cleanup
A slimmer image may reduce the startup time in openstack.  To find potential areas to cleanup, I launched the image and looked for some directories that consumed the most storage.  I found that the /usr/lib/modules directory contained modules for two kernels, the first kernel left over from before the upgrade of the image.  I added the following to the cleanup script that packer runs before creating the final image.

{% highlight shell %}
# Remove old kernels
package-cleanup -y --oldkernels --count=1
{% endhighlight %}

After compression, that cleared 19MB out of the image.
{% highlight console %}
670M    test.RHEL7.grub.qcow2
651M    test.RHEL7.1kernel.qcow2
{% endhighlight %}

This will remove all but the newest kernel and the related modules and reduced the create time to 16 seconds.

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Image create  | 17s | 37% | 16s | 5% | 57%
| OS Boot       | 69s| 63% | 8s | 82% | 33%

## virt-sparsify
As part of the packer process for provisioning a script called zerodisk.sh is executed, that empties out free space. Here is what that script looks like:
{% highlight shell %}
#
# Zero out the free space to save space in the final image:
#
dd if=/dev/zero of=/EMPTY bs=1M
rm -f /EMPTY /tmp/script.sh
sync
{% endhighlight %}

Historically, this has saved about 400 MB in the compressed image size.  However, some further optimizations are capable by using [virt-sparsify](http://libguestfs.org/virt-sparsify.1.html).  virt-sparsify is part of the libguestfs suite of tools that contains several nice programs for managing and optimizing virtual machines.

Running sparsify saves close to 60 MB.
{% highlight console %}
651M    RHEL7.20161130.qcow2
599M    RHEL7.20161130.sparse.compress.qcow2
{% endhighlight %}

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Image create  | 17s | 37% | 16s | 5% | 57%
| OS Boot       | 69s| 63% | 8s | 82% | 33%

No improvement in create time, but we saved some space, so we will keep it.

## virt-sysprep
Just like for zeroing the disk, a script was used to cleanup and prepare the VM.  This would remove certain files and clear out DHCP addresses as discussed in the [previous post]({% post_url 2016-11-29-bluebox-optimize-boot %}).  To further optimize image create, [virt-sysprep](http://libguestfs.org/virt-sysprep.1.html) is used to clean out more items than homegrown scripts.

I started with the following options for cleanup.
{% highlight console %}
virt-sysprep --enable abrt-data,bash-history,blkid-tab,ca-certificates,crash-data,cron-spool,dhcp-client-state,dhcp-server-state,dovecot-data,firewall-rules,firstboot,flag-reconfiguration,hostname,kerberos-data,logfiles,machine-id,mail-spool,net-hostname,net-hwaddr,pacct-log,package-manager-cache,pam-data,password,puppet-data-log,random-seed,rhn-systemid,rpm-db,samba-db-log,script,smolt-uuid,ssh-hostkeys,ssh-userdir,sssd-db-log,tmp-files,udev-persistent-net,user-account,utmp,yum-uuid -a "$1"
{% endhighlight %}

Running virt-sysprep saves another 10MB on the image size
{% highlight console %}
590M    RHEL7.20161130.sparse.compress.qcow2
{% endhighlight %}

This saved some space and helped us remove some unnecessary cleaning code in our packer setup, but did not improve create time to any measurable degree.

## Optimizing the Kitchen Openstack driver
With the image fully optimized, other areas of optimization required investigation.

I reached out to support at Bluebox and asked for help.  They turned on debugging in Openstack and I ran converge tests.  Bluebox was launching an image in under 7 seconds.  What was consuming the other 9 seconds?  I enabled debug in kitchen and in the underlying fog gem used by kitchen-openstack driver to talk to Bluebox.  This additional debugging showed that a lot of the time is spent in getting authentication tokens and performing lookups to convert named entities such as image, network, and flavor into openstack UUIDs.  To optimize this part, I create a fork of kitchen-openstack driver which let me explicitly set the UUIDs of networks, flavor and image.  This reduced create time down to 11s.

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Image create  | 17s | 37% | 11s | 35% | 57%
| OS Boot       | 69s| 63% | 8s | 82% | 33%

I am awaiting approval to contribute to kitchen-openstack before I submit a PR for my enhancement.


# Summary
With image optimization complete, a new image is now ready in about 20 seconds.  

| Stage         | Duration (seconds)     | % Total  |
| ------------- | ------------- | ----- |
| Create        | 20 seconds  | 1% |
| Prepare       | 7 minutes 36 seconds     |   41% |
| Converge      | 10 minutes 14 seconds       |   67% |
| Verify        | 11 seconds      |    1% |

The approach we will take is to optimize the testing process in the order of the kitchen stages.  Next up, will be kitchen prepare optimization.

# Questions?
[Reach out to me on twitter](https://twitter.com/boc_tothefuture)
