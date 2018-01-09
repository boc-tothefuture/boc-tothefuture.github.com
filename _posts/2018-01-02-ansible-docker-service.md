---
title:  "Managing Docker containers as services with Ansible"
date:   2018-01-02 08:00:00 -0400
categories: ansible docker container service
excerpt: "Managing Docker containers as services with Ansible"
---

<br>
![Bluebox](/images/docker/docker.png)
<br>
<br>

# Homelab
Over the holiday break, I was able to spend some time optimizing my homelab that runs my automation, entertainment and other household support functions. I had been managing most of the systems by hand. As I move to containerize some of my services for an upcoming system upgrade I took the opportunity to learn some Ansible and use that to begin managing my homelab.

My homelab consists of two hosts.  One the storage host runs [OmniOS](https://omnios.omniti.com/wiki.php/WikiStart), the other is a VMWare ESXi host that runs various linux and windows VMs.  

# Docker containers for services
I had been manually managing two docker services, [SageTV](https://github.com/google/sagetv) and [Unifi](https://hub.docker.com/r/jacobalberty/unifi/). I had struggled to find a reliable method to start these on boot, restart them when necessary, and in general management of these containers. Setting these containers up on my homelab was my initial exposure to docker and it showed.

# systemd and Docker
I wanted to manage my containers as if they were system services.  This would bring familiar start/stop actions, as well as ensuring they were started on boot.  [This article](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/managing_containers/using_systemd_with_containers) has good background information on how to manage docker containers with systemd.

# systemD and Docker Playbook
I started off with [Manual Hutter's](https://github.com/mhutter) [ansible playbook](https://galaxy.ansible.com/mhutter/docker-systemd-service/) for managing docker containers with systemd and modified it for my environment. I made two changes.  

1. I removed the task where the service pulled latest using the command module on every run. I wanted to control the version installation of the docker container using the native docker_image module in ansible.
2. I had to rename the "name" variable to "service_name". In version 2.4 of ansible there is a [defect](https://github.com/ansible/ansible/issues/30776) that passes through the name of the included role rather than the variable.

# Steps

1. Create a generic docker role for my lab
2. Create a role for each container I want to manage


# Docker playbook
I created a docker playbook that would configure my host system to support the docker containers I planned to manage.  This playbook ensures the docker and nfs packages are installed, installs pip, and the docker-py python package and then ensures the containers storage directory on the storage host is mounted.

{% highlight yaml %}
---
- name: Install packages
  apt:
    name: "{{ item }}"
    cache_valid_time: 86400
    state: present
  with_items:
    - docker
    - nfs-common

- name: Installed pip
  easy_install:
    name: pip
    state: latest

- name: Installed docker-py
  pip:
    name: docker-py
    state: latest

- name: Mount containers storage
  mount:
  {% raw %}path: "{{ containers_mount }}" {% endraw %}
    src: "storage:/tank/containers"
    fstype: nfs
    state: mounted
{% endhighlight %}

# Configure and start docker container as a service
Several steps are necesary in the playbook to setup the docker service.

1. Call our base docker role to setup docker
2. Validate the storage directory exists for this particular container
3. Pull a specific docker image
3. Use mhutter's modified ansible role to create the appropriate services, enable and start it.

Below the playbook that performs those steps for the SageTV docker container.

{% highlight yaml %}
--
- name: Enable docker
  import_role: name=docker

- name: Ensure storage directory exists
  file:
    path: "{{ containers_mount }}/sagetv"
    state: directory

- name: Pull sagetv image
  docker_image:
    name: stuckless/sagetv-server-java8

- name: Create sagetv docker service
  include_role:
    name: mhutter.docker-systemd-service
  vars:
    service_name: sagetv
    image: stuckless/sagetv-server-java8:latest
    volumes:
      - "/mnt/recordings/:/var/media"
      - "/mnt/movies/:/var/mediaext"
     {% raw %} - "{{ containers_mount }}:/opt/sagetv" {% endraw %}
    args: >
      --net host
      --env PUID=1000
      --env PGID=1000
      --env VIDEO_GUID=19
      --env OPT_GENTUNER=N
      --env OPT_COMMANDIR=N
      --env OPT_COMSKIP=Y
      --env JAVA_MEM_MB=768
      --env VERSION=latest
      --env OPT_SETPERMS=Y
      --privileged
      --interactive
{% endhighlight %}

# Service

After the playbook executes, the SageTV container is ready to go.

{% highlight shell %}
systemctl status sagetv_container
● sagetv_container.service
   Loaded: loaded (/etc/systemd/system/sagetv_container.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2017-12-30 22:05:04 EST; 2 days ago
 Main PID: 7168 (docker)
    Tasks: 13
   Memory: 5.5M
      CPU: 7.798s
   CGroup: /system.slice/sagetv_container.service
           └─7168 /usr/bin/docker run --name sagetv --rm -v /mnt/recordings/:/var/media -v /mnt/movies/:/var/mediaext -v /mnt/containers/sagetv/:/opt/sagetv -p 7080:8080 --net host --env PUID
{% endhighlight %}

# Summary

Reusing the ansible service role and a few lines of YAML makes it easy to install, configure and run docker containers as services.  


# Questions?
[Reach out to me on twitter](https://twitter.com/boc_tothefuture)
