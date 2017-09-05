---
title:  "Optimize kitchen converge Chef"
date:   2017-08-24 12:00:00 -0400
categories: openstack chef
excerpt: "Optimize kitchen converge with remote testing"
---

<br>
![Bluebox](/images/bluebox/converge.jpg)
<br>
<br>

# Optimizing remote testing
A [previous article]({% post_url 2017-08-06-optimize-prepare %}) described how to optimize the prepare phase of kitchen testing. This post will focus on the kitchen converge step.

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

<br>
# Kitchen Converge
The converge stage of test kitchen executes the specified chef recipes on the test node.  This is the stage that should, in most instances, take the longest. Converge has two distinct segments:
*  Cookbook resolution
*  Chef Run

## Cookbook resolution
This stage resolves all of the cookbooks and their dependencies.  Depending on the number of dependencies this can consume up to 60 seconds of the converge run. I may attempt to optimize the stage at a later time, but for now I have not made any changes.

## Chef run optimization strategy
It is not feasible to cover every potential optimization path for all cookbooks as those optimizations are heavily dependent on what the cookbook is trying to accomplish.  Below are several generic optimization recommendations.

## Package everything
I recommend using [fpm-cookery](https://github.com/bernd/fpm-cookery) to create system native package for everything installed by Chef. This includes repackaging of complex binary installers, arbitrary small binary file like security keys, and system packages. Basically, anything that is not a configuration file should be contained in a system package. This speeds up binary installers, which sometimes need a jvm to start, etc and prevents having to write complex custom guards to determine if actions are necessary. Additionally, for YUM based systems, after the initial caching of the YUM data Chef can very quickly make decisions if the package is already installed at the right level.

### FPM Cookery Example
Below is a quick example of using FPM cookery to build a system package for a product that initially required a more complex Chef resource to manage. The examples consists of two parts, a [vagrant](https://www.vagrantup.com/) file for creating a virtual machine and the [fpm-cookery](https://github.com/bernd/fpm-cookery) recipe.

#### Vagrant File
Below is a sample Vagrant File to use in FPM Cookery

{% highlight ruby %}
# -*- mode: ruby -*-
# vi: set ft=ruby :

RHEL_PLATFORMS = %w(6 7).freeze

def fpm_script(rhel_version)
  <<-SCRIPT
mkdir -p /tmp/fpm-temp
mkdir -p /tmp/fpm-cache
cd build/recipes
sudo fpm-cook clean --tmp-root /tmp/fpm-temp --cache-dir /tmp/fpm-cache --pkg-dir /home/vagrant/build/pkg/#{rhel_version}
sudo fpm-cook install-deps
sudo fpm-cook package --tmp-root /tmp/fpm-temp --cache-dir /tmp/fpm-cache --pkg-dir /home/vagrant/build/pkg/#{rhel_version}
SCRIPT
end

Vagrant.configure(2) do |config|
  RHEL_PLATFORMS.each do |rhel_version|
    config.vm.define box_name(rhel_version) do |build|
      build.vm.box = box_name(rhel_version)

      build.vm.provider 'virtualbox' do |vb|
        vb.customize ['modifyvm', :id, '--memory', '2048']
        vb.linked_clone = true
      end
      build.vm.synced_folder 'recipes/', '/home/vagrant/build/recipes'
      build.vm.synced_folder 'stage/', '/home/vagrant/build/stage'
      build.vm.synced_folder 'pkg', '/home/vagrant/build/pkg/'

      build.vm.provision 'chef_solo' do |chef|
        chef.add_recipe 'ei-build'
        chef.add_recipe 'ei-java'
      end
      build.vm.provision 'shell', inline: fpm_script(rhel_version)
    end
  end

  config.berkshelf.enabled = true
end
{% endhighlight %}

The above vagrant file creates a new RPM for both RHEL 6 and 7 platforms.  The work of calling FPM cookery to create the RPM is done by the script definition at the top.  The vagrant configuration iterates over both the RHEL 6 and 7 platform, creates a new virtual machine, links the appropriate directories, executes two sets of chef recipes the ei-build cookbook that installs most dependencies for building images and a java installation which is required for this particular build.  After the dependencies are installed, the fpm_script defined above is invoked.

#### FPM Cookery Recipe
Below is an example FPM Cookery recipe

{% highlight ruby %}
class UcdAgent < FPM::Cookery::Recipe
  source File.join('/home/vagrant/build/stage/', 'ibm-ucd-agent.zip'), :with => :local_path
  omnibus_package true
  omnibus_dir '/opt/ibm-ucd/agent'

  name 'ucd-agent'
  version '1:6.2.4.0'
  revision '3'
  arch 'x86_64'
  vendor 'ei'
  description 'IBM Urbancode Deploy Agent'
  config_files '/opt/ibm-ucd/agent/conf/agent/installed.properties'
  replaces 'IBMUrbancodeDeployAgent'
  depends 'ei-java-cryptography-extension'

  def build
    FileUtils.chmod 0755, Dir.glob(File.join(builddir, ' /opt/ibm-ucd/agent/opt/udclient/udclient'))
  end

  def install
    install_props_source = File.join(workdir, 'ei.agent.install.properties')
    install_props_dest = File.join(builddir, 'ibm-ucd-agent', 'ibm-ucd-agent-install')
    FileUtils.cp(install_props_source, install_props_dest)
    environment.with_clean { safesystem('sudo ./ibm-ucd-agent-install/install-agent-from-file.sh ./ei.agent.install.properties') }
  end
end
{% endhighlight %}

This is a multi-platform recipe used to create the RPM for the Urbancode Deploy Agent.  FPM Cookery, and the underlying [FPM](https://github.com/jordansissel/fpm) application abstract away the usually painful process of creating packages.  This is all the code necessary to build the package.  The recipe has a few distinct sections.  The very first sections are a DSL in FPM-Cookery that describe inforation about how to make the paackage and some metadata for when the package is created.  The source section, tells FPM-Cookery to grab the file in that path and uncompress it.  The omnibus directives tell FPM-Cookery to use the contents of the specified directory as the location to put into the package.  The name through depends identifiers are metadata that is passed onto the package itself.  Next, FPM-Cookery calls the build and install methods.  The build methods changes the permissions on files.  The install method prepares a properties file to be used with the installer and then calls the installer.  This executes the actual install.  When it is complete, FPM-Cookery calls FPM to package up the contents of /opt/ibm-ucd-agent and the package is created.

It only takes a handful of lines of code to repeatably create packages.

What are the benefits of doing this? Our environment uses UrbanCode deploy on every node.  Every Chef role converge would need to install this package. How much time does this save?  Running the install above via a chef resource took about 30 seconds, the equivalent yum install takes less than 2 seconds. Some of larger packages like IBM Tivoli Monitoring went from 5+ minutes to under 50 seconds.


## yum-cache
Many recipes will install packages, if they system is RedHat - YUM is the package manager of choice.  When Chef encounters a package statement, it uses yum to install those package.  The first thing yum will do is download all yum repo data and then create the cache.  

{% highlight bash %}
/usr/bin/time -f "%E elapsed" yum makecache

...

Metadata Cache Created

0:14.95 elapsed
{% endhighlight %}

That operation can take close to 15 seconds to complete.  
You can take advantage of the fact that the system is started while test-kitchen is uploading cookbooks in the prepare stage to pre-warm the YUM cache.  This is best done using the 'at' command inside of cloud-init.  The reason to use at is that the at job will immediately exit and run the job in the background.  If 'at' was not used cloud-init would delay the boot process until the yum cache was created, negating any potential gains.

The cloud init configuration looks like this:

{% highlight bash %}
write_files:
  - path: /tmp/yum_makecache
    owner: root:root
    permissions: '0644'
    content: |
     # Create the yum cache - this will run in parallel while SSH is starting and chef is transferring cookbooks, etc.
     yum makecache


runcmd:
  - 'at now -M -f /tmp/yum_makecache'

{% endhighlight %}

This update to the cloud-init file saves about 15 seconds off every converge.


# Summary

Package optimization and yum-cache enhancements provide a measurable performance improvement in the converge phase:

| Stage         | Original Duration | Original % Total  | Optimized Duration| % Improvement | Optimized % total
| ------------- | ------------- | ----- | ---- | ------ | ----- |
| Kitchen Converge  | 10 minutes 14 seconds | 95% | 6 minutes 59 seconds | 31.76% | 92%

With converge optimization complete, 15 seconds is reduced from every converge by using the cloud-init optimization for yum cache creation. Using packages through the fpm-cookery method saves additional time. One item, that is not captured quantitatively, is the move to remote testing will often help with data-locally.  In this example, all packages are stored in Object Storage which is local to the cloud region, enable rapid download of all packages, that helps with converge time.

| Stage         | Duration (seconds)     | % Total  |
| ------------- | ------------- | ----- |
| Create        | 20 seconds  | 1% |
| Prepare       | 3.4 seconds     |   1% |
| Converge      | 6 minutes 59 seconds       |   92% |
| Verify        | 11 seconds      |    2% |

Next up for optimization? Verify.

# Questions?
[Reach out to me on twitter](https://twitter.com/boc_tothefuture)
