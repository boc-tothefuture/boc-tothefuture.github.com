---
title:  "Simple Webserver with Openstack Swift"
date:   2016-09-17 12:00:00 -0400
categories: swift openstack webserver
excerpt: "Using Openstack Swift as a webserver"
storage_services:
  - url: openstack/storage_services.jpg
    image_path: openstack/storage_services.jpg
    alt: "Storage services on Bluemix"
    title: "Storage services on Bluemix"
storage_credentials:
  - url: openstack/storage_credentials.jpg
    image_path: openstack/storage_credentials.jpg
    alt: "Storage credentials on Bluemix"
    title: "Storage credentials on Bluemix"
add_container:
  - url: openstack/add_container.jpg
    image_path: openstack/add_container.jpg
    alt: "Add a container"
    title: "Add a container"
name_container:
  - url: openstack/name_container.jpg
    image_path: openstack/name_container.jpg
    alt: "Name the container"
    title: "Name the container"
add_files:
  - url: openstack/add_files.jpg
    image_path: openstack/add_files.jpg
    alt: "Add files to a container"
    title: "Add files to a container"
files_added:
  - url: openstack/files_added.jpg
    image_path: openstack/files_added.jpg
    alt: "Files added to container"
    title: "Files added to container"

---

<br>
![Openstack](https://dal.objectstorage.open.softlayer.com/v1/AUTH_2f1d538663bf4333b8bb498369241ce5/foo/openstack.jpg)
<br>
<br>

# Swift Based Webserver

> How can I use Swift as a simple websever?

I received this question a few times recently.  Below is a quick how to using [Bluemix](http://www.ibm.com/BlueMix) object storage.


## Steps

1. Create a swift instance
2. Put content into swift
3. Permit unauthenticated read access to the container

<br>

# Create a swift container

Login into your BlueMix account [here](https://console.ng.bluemix.net/).  After you login, select catalog from the top navigation. On the left navigation bar select the checkbox next to Storage to limit the display to IBM Cloud storage services only.  Your screen should look like this:

{% include gallery id="storage_services" caption="List of storage services on Bluemix" %}

* Select Object Storage
* Select whichever plan is necessary for your scaling goals
* Leave the other defaults values
* Click create.


After a few moments, the IBM Cloud Object Storage service should be created and added to your Bluemix space.  

Click on service credentials on the left navigation bar to find your API key.

Note this API key, you will need it to access your object storage instance.
{: .notice--info}

{% include gallery id="storage_credentials" caption="Storage credentials" %}

Your Bluemix space now has object storage enabled

# Insert content into swift

## Create a container
Click on "Manage" on the left navigation bar. Then click "Add a container"

{% include gallery id="add_container" caption="Add storage container to Bluemix" %}

## Name the container
After clicking "Add a container" you will be prompted to give the container a name.  Choose whatever name you want.

{% include gallery id="name_container" caption="Name your new storage container in Bluemix" %}

## Add a file
Click on "Add files" to upload new files to your container.  This will open a file selector window.  I uploaded the Openstack image you see above.

{% include gallery id="add_files" caption="Add a file to your storage container in Bluemix" %}

Once you select your file, your container information in Bluemix should update with storage space consumed and details on the uploaded file.  

{% include gallery id="files_added" caption="Files added to to your storage container in Bluemix" %}


# Permit unauthenticated read access to the container

## Setup swift command line tool
Follow the steps in the [Bluemix Docs](https://console.ng.bluemix.net/docs/services/ObjectStorage/objectstorge_usingobjectstorage.html#using-swift-cli) to setup and install the swift CLI.

## Export service credentials
Once the CLI is installed the first step will be to take the service credentials you noted earlier and export them into the appropriate environment variables for the swift tool.  Below is an example.

{% highlight shell %}
export OS_USER_ID=24a20b8e4e724f5fa9e7bfdc79ca7e85
export OS_PASSWORD=aaa55AAAaaaaa]?,
export OS_PROJECT_ID=383ec90b22ff4ba4a78636f4e989d5b1
export OS_AUTH_URL=https://identity.open.softlayer.com/v3
export OS_REGION_NAME=dallas
export OS_IDENTITY_API_VERSION=3
export OS_AUTH_VERSION=3
{% endhighlight%}

## Authenticate
Next execute the "swift auth" command.  This will authenticate the swift session and output the storage path to the files that were added.
{% highlight shell %}
swift auth
export OS_STORAGE_URL=https://dal.objectstorage.open.softlayer.com/v1/AUTH_2f1d538663bf4333b8bb498369241ce5
export OS_AUTH_TOKEN=fooooooovlYmo5-UdEdYg_uRh0JWniXxKr_zuO0FC__ebrXqrhOPwVZRwa-EBPTbrTd1fwQ7-Dhyqs7ZZsc2uEtrmvwD6g5Wi3Vg__tnl1LkfClsY6rR_iZgYFcDPwYdsLbpCSo0eoU7FNsZHTCJyXJefZXhzm2aWZC0XPMeaDoFv0uik
{% endhighlight%}

Note the OS_STORAGE_URL, this is the base URL for all unauthentacted public requests to object storage.
{: .notice--info}

## Modify container permissions
Use the swift command to enable unauthenticated read access to the container.

{% highlight shell %}
swift post foo --read-acl ".r:*,.rlistings"
swift post -m 'web-listings: true' foo
{% endhighlight%}

## Test access

Noting the OS_STORAGE_URL above, the fully constructed URL for my object is:
{% highlight shell %}
https://dal.objectstorage.open.softlayer.com/v1/AUTH_2f1d538663bf4333b8bb498369241ce5/foo/openstack.jpg
{% endhighlight%}

Test accessing this with wget:
{% highlight html %}
wget -O/dev/null -q https://dal.objectstorage.open.softlayer.com/v1/AUTH_2f1d538663bf4333b8bb498369241ce5/foo/openstack.jpg && echo exists || echo not exist
exists
{% endhighlight%}

If the above command outputs exists, the file is now readable by all.  Any additional files you add to the container will now also be readable from the web.

# Limitations
You can use use Bluemix object storage as a quick and dirty webserver.  However, object storage does not give you the full control and customizations usually needed for a production web server.  For simple things this is an easy way to get objects onto the web.  In fact, I am serving the openstack logo at the top of this page out of Bluemix object storage.

# Resources
Should you run into any issues consult the [BlueMix Documentation](https://console.ng.bluemix.net/docs/) or [reach out to me on twitter](https://twitter.com/boc_tothefuture)
