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
