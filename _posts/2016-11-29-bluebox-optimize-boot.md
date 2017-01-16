---
title:  "Optimize image boot in Bluebox with Chef"
date:   2016-12-07 12:00:00 -0400
categories: openstack chef Bluebox
excerpt: "Optimize image booting in Bluebox for use with Test Kitchen"
---

<br>
![Bluebox](/images/bluebox/boot.jpg)
<br>
<br>

# Optimizing remote testing
A [previous article]({% post_url 2016-11-28-bluebox-optimizing %}) broke down the goals and an optimzation approach.  This post will describes the steps taken to optimize the boot part of the create stage.

As a reminder, here is the current duration for each stage.

| Stage         | Duration (seconds)     | % Total  |
| ------------- | ------------- | ----- |
| Create        | 1 minute 22 seconds  | 8% |
| Prepare       | 7 minutes 36 seconds     |   39% |
| Converge      | 10 minutes 14 seconds       |   52% |
| Verify        | 11 seconds      |    1% |


# Previous Articles

1. [Remote testing in Bluebox with Chef]({% post_url 2016-11-19-bluebox-testing %})
1. [Optimization overview of Bluebox with Chef]({% post_url 2016-11-28-bluebox-optimizing %})



# Create sub-stages
The test kitchen create finishes when the newly provisioned instance can be logged into.  In the previous article, the test process was broken down into distinct stages.  In this article, the create stage will be broken down into distinct sub-stages for optimization.

The two stages to look at are image create time in Bluebox/Openstack and then operating system boot time.  Test kitchen has no visibility into the boot process, therefore to calculate the sub stage times, we will have to compare the log timestamp for the node create request and correlate it with the timestamps in /var/log/message

Nova create request
{% highlight console %}
I, [2016-11-29T13:24:38.901495 #32083]  INFO -- cmusta-cdt-RHEL7: -----> Creating <cmusta-cdt-RHEL7>...
{% endhighlight %}

Start of /var/log/message
{% highlight console %}
Nov 29 18:25:18 localhost rsyslogd: [origin software="rsyslogd" swVersion="7.4.7" x-pid="626" x-info="http://www.rsyslog.com"] start
{% endhighlight %}

Boot complete
{% highlight console %}
Nov 29 18:26:27 localhost systemd: Startup finished in 612ms (kernel) + 1.093s (initrd) + 1min 9.960s (userspace) = 1min 11.667s.
{% endhighlight %}

After normalizing for timezones.

| Stage         | Duration | % Total  |
| ------------- | ------------- | ----- |
| Image create  | 40s| 37% |
| OS Boot       | 69s| 63% |

Because OS Boot takes longer than image create, optimizations will first be applied the boot process.   

# OS Boot Optimization
systemd-analyze is a great tool to optimize the boot of systemd based systems.

{% highlight console %}
systemd-analyze blame | head
        51.662s cloud-init.service
        15.150s NetworkManager-wait-online.service
         1.598s kdump.service
          867ms lvm2-monitor.service
          842ms cloud-init-local.service
          812ms systemd-udev-settle.service
          762ms lvm2-pvscan@252:2.service
          708ms postfix.service
          631ms network.service
          613ms dev-mapper-rootvg\x2drootlv.device
{% endhighlight %}

Most of the boot time is spent in cloud-init and waiting for the Networkmanager to finish initialization.

## DNS
The system and cloud init logs recorded multiple timeouts.  A previous commit showed that DNS management by NetworkManager had been disabled.  This worked under VirtualBox, but not with Bluebox because this Bluebox testing setup needs to use the internet for tests rather than internal DNS servers.

This line was removed from the scripts run by packer to prepare the image.
{% highlight console %}
# Stop NetworkManager from updating /etc/resolv.conf
echo -e "[main]\ndns=none" >/etc/NetworkManager/conf.d/99-dns.conf
{% endhighlight %}

After that change, OS Boot time was highly improved.  

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Image create  | 40s| 37% | 47s | -17% | 65%
| OS Boot       | 69s| 63% | 25s | 64% | 35%

The OS boot time decreased 44 seconds (63.77%) decrease in boot time and now only 35% of the total boot time.  The image create increased a little bit, but I expect that to vary due to caching and other variables.

systemd-analyze tells the story quite clearly, fixing DNS took cloud-init from 51 seconds to 1.7 seconds.
{% highlight console %}
systemd-analyze blame | head
         17.012s NetworkManager-wait-online.service
          1.764s cloud-init.service
          1.741s kdump.service
          1.437s cloud-init-local.service
           972ms systemd-udev-settle.service
           778ms postfix.service
           666ms network.service
           599ms lvm2-monitor.service
           535ms cloud-config.service
           526ms cloud-final.service
{% endhighlight %}

The NetworkManager wait online seems to be taking longer than one would expect.

## DHCP
Looking at the network startup using systemd, there is an interesting 16 second delay for networking to complete that seems to be the time it takes for DHCP to acquire a DHCP lease.
{% highlight console %}
Nov 29 19:31:49 localhost dhclient[682]: DHCPREQUEST on eth0 to 255.255.255.255 port 67 (xid=0x960847f)
Nov 29 19:31:56 localhost dhclient[682]: DHCPREQUEST on eth0 to 255.255.255.255 port 67 (xid=0x960847f)
Nov 29 19:32:05 localhost dhclient[682]: DHCPDISCOVER on eth0 to 255.255.255.255 port 67 interval 3 (xid=0x520d7ae2)
Nov 29 19:32:05 localhost dhclient[682]: DHCPREQUEST on eth0 to 255.255.255.255 port 67 (xid=0x520d7ae2)
Nov 29 19:32:05 localhost dhclient[682]: DHCPOFFER from 192.168.0.2
Nov 29 19:32:05 localhost dhclient[682]: DHCPACK from 192.168.0.2 (xid=0x520d7ae2)
{% endhighlight %}

This is a long time to wait for a lease.  [wikipedia](https://en.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol) indicates that the order of operations in the log wrong.  A request is being sent, timed out, then again, then finally a discovery.  This is indicative of DHCP trying to obtain a previous address which the server is ignoring.  After some research, it appears the DHCP address is not cleared out of the image.  The following was added to the cleanup script run by packer.

{% highlight console %}
# Remove all dhcpd related items
rm -vrf /var/lib/dhclient/*
rm -vrf /var/lib/dhcp/*
rm -vf /var/lib/NetworkManager/dhclient-\*.lease
{% endhighlight %}

Fixing DHCP again improved OS boot.

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Image create  | 40s| 37% | 37s | 8% | 71%
| OS Boot       | 69s| 63% | 15 | 78% | 29%

systemd-analyze once again helps tells the story, Fixing the DHCP timeout saved 10 seconds during boot.
{% highlight console %}
systemd-analyze blame | head
         5.426s cloud-init.service
         4.995s NetworkManager-wait-online.service
         3.043s kdump.service
         2.079s postfix.service
          860ms lvm2-monitor.service
          852ms systemd-udev-settle.service
          783ms cloud-init-local.service
          692ms cloud-config.service
          641ms lvm2-pvscan@252:2.service
          555ms network.service
{% endhighlight %}


## Grub
Comparing the create timestamp to the actual boot, there was a delay before any messages show up in the /var/system/log.  This delay matches up exactly with the grub boot timeout.  

Grub2 defaults
{% highlight shell %}
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .\*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=rootvg/rootlv rd.lvm.lv=rootvg/swaplv rhgb quiet"
GRUB_DISABLE_RECOVERY="true"
{% endhighlight %}

The timeout was changed to zero and a new image was built and tested.

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Image create  | 40s| 37% | 32s | 54% | 71%
| OS Boot       | 69s| 63% | 13s | 81% | 29%


While this is an image optimization, the improvement shows up in the image create section because it reduces the time between image create request and the first log message in /var/log/messages.  The total time for the create stage is down to 45 seconds.  Almost half the original measurement.

## Remove unnecessary services
Inspecting systemd-analyze blame a few services pop up as potentially causing some slowness during boot.  
{% highlight console %}
systemd-analyze blame | head
         5.426s cloud-init.service
         4.995s NetworkManager-wait-online.service
         3.043s kdump.service
         2.079s postfix.service
          860ms lvm2-monitor.service
          852ms systemd-udev-settle.service
          783ms cloud-init-local.service
          692ms cloud-config.service
          641ms lvm2-pvscan@252:2.service
          555ms network.service
{% endhighlight %}

NetworkManager-wait-online and cloud-init.service is a candidate for optimization.  The NetworkManager-wait-online service is important in some situations to prevent the system from booting before the network is fully online.  In this particular use case, sshd is well behaved enough to not need it.

Prior to uploading the image, we use virt-customize to mask the NetworkManager-wait-online service to remove it from the startup.
{% highlight console %}
#Disable wait online for test nodes
virt-customize --no-network --run-command 'systemctl mask NetworkManager-wait-online.service' -a "$1"
{% endhighlight %}

## Optimize cloud-init
The cloud-init service is averaging around 5.5 seconds during boot.  Inspecting, the default config, cloud-init uses the EC2 service provider which is emulated in OpenStack.  Switching that to use the 'OpenStack' provider further optimized the boot time reducing the cloud-init.service to a 2 second start.

# Summary

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Image create  | 40s| 37% | 32s | 54% | 80%
| OS Boot       | 69s| 63% | 8s | 88% | 20%


Removing services and optimizing cloud-init reduced boot time to 8.4 seconds.  This is a 88% improvement from the start point.  The next article will focus on optimizing the image create portion of the kitchen create stage.

# Questions?
[Reach out to me on twitter](https://twitter.com/boc_tothefuture)
