---
layout: post
title: "TYPO3 Virtual Appliance with Packer"
description: ""
category: partheas
tags:
  - php
  - typo3
  - automation
  - development
---
{% include JB/setup %}

We are currently working on a [TYPO3](http://typo3.org) project with a small team
of 2 developers and 1 designer. To avoid issues everyone obviously needs to work
on the same TYPO3 version, same PHP version, same MySQL version, etc.

Initially I thought of going for [Vagrant](http://vagrantup.com), but just as I intended to work on it this
tweet passed by in my timeline:

<blockquote class="twitter-tweet"><p>I'm excited to announce Packer: an open source tool for building machine images. <a href="http://t.co/2wwY68GHTU">http://t.co/2wwY68GHTU</a> <a href="http://t.co/Yx8ImYvSGK">http://t.co/Yx8ImYvSGK</a></p>&mdash; Mitchell Hashimoto (@mitchellh) <a href="https://twitter.com/mitchellh/statuses/350647179729309696">June 28, 2013</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

**Why [Packer](http://packer.io) instead of Vagrant?** Because it supports much more.
I can use it for plain Virtual Machines (useful for non-developers), use it to create
production servers and use it for development as Vagrant boxes! In this particular
use case I will use the same Packer template to create a [Digital Ocean](https://www.digitalocean.com)
droplet for the staging environment as well as for Vagrant boxes to be used by the developers.

Besides having the same environment everywhere I also wanted to fully automate the
TYPO3 installation without manually having to go through the setup wizard, so I wrote
a little [script](https://gist.github.com/jerser/de36a686f3591d4f91d8) with
[CasperJS](http://casperjs.org) to automate that.

Currently all it does is executing the 1-2-3 setup, but in the near future I want
to use this approach for deployment automatisation and configuration of TYPO3
so everything is scripted and thus repeatable by everyone. This should greatly
reduce risks between developers and development/staging/production.

### Setup

* CentOS 6.4 (minimal cd)
* MySQL (latest version in default CentOS repo)
* PHP 5.4.16
* TYPO3 6.1.1

Whenever possible, the provisioning/template does not use versions but uses the latest
version, the goal is to automatically rebuild when a new version of TYPO3 or one of
its dependencies is released.

#### Booting the machine + running setup

The installation is [kickstarted](https://www.centos.org/docs/5/html/Installation_Guide-en-US/s1-kickstart2-file.html)
by this [Anaconda kickstart file](https://github.com/jerser/packer-templates/blob/master/kickstart/centos_minimal.cfg)
served by Packers built-in webserver.

I just want to clarify one piece out of the kickstart file. I had to create an init
script that is only executed on the first boot to be able to pull in updated packages.
I first tried it directly in the post part of the kickstart but this failed because
there is only a local network connection (over which the kickstart file is served).
The script also reboots after updating to make sure that any new kernel is also loaded
before provisioning from inside Packer.

{% highlight bash %}
# Create first-boot script to run only once
%post

cat > /etc/init.d/firstboot <<EOF
#!/bin/bash
#
# firsboot Run some post-installation tasks
#
# chkconfig: 2345 15 20
# description: Updates on first boot, after networking and before ssh \
#              and then reboot again to reload kernel changes
#

case "\$1" in
  start)
    yum update -y --skip-broken
    chkconfig firstboot off
    init 6
  ;;
  stop|status|restart|reload|force-reload)
    # do nothing
  ;;
esac
EOF
chmod +x /etc/init.d/firstboot
chkconfig firstboot on

exit 0

%end
{% endhighlight %}

#### Provisioning

I started out doing provisioning via the inline commands option, but this quickly
becomes a mess, so I have now moved everything to separate provisioning scripts.
You can see all scripts [here](https://github.com/jerser/packer-templates/tree/master/provisioning).

The [Packer documentation](http://www.packer.io/docs/templates/provisioners.html) mentions an interesting overrides option for provisioning
that should allow separate commands/scripts to be executed based on the active builder (e.g.: virtualbox).

Unfortunately I didn't get it to work (or misunderstood the documentation), so I
went ahead and installed [virt-what](http://people.redhat.com/~rjones/virt-what/) so I can
check in a shell script if it is running inside VirtualBox or not. This is important
because obviously you do not want to install the VirtualBox Guest Additions in
non-VirtualBox images.

{% highlight bash %}
#!/bin/bash

yum -y install virt-what

if [ `virt-what` == "virtualbox" ]; then
  yum -y install kernel-devel kernel-headers gcc make
  rpm -Uvh https://dl.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm
  yum -y install dkms
  export KERN_DIR=/usr/src/kernels/`uname -r`
  mkdir /mnt/VBoxGuestAdditions
  mount -o loop VBoxGuestAdditions.iso /mnt/VBoxGuestAdditions
  /mnt/VBoxGuestAdditions/VBoxLinuxAdditions.run
  umount /mnt/VBoxGuestAdditions
  rmdir /mnt/VBoxGuestAdditions
fi

rm -f VBoxGuestAdditions.iso
{% endhighlight %}

### Try it!

You can get a zip of the files [here](https://github.com/jerser/packer-templates/archive/master.zip)
or check the project on [GitHub](https://github.com/jerser/packer-templates).

#### How to use it?

    packer build typo3/typo3.json

    ==> Builds finished. The artifacts of successful builds are:
    --> virtualbox-typo3: VM files in directory: output-virtualbox-typo3
    --> virtualbox-typo3: 'virtualbox' provider box: packer_virtualbox.box

When the build is finished you end up with one directory containing a VirtualBox
configuration file and a VirtualBox harddrive (output-virtualbox-typo3) and one
other file packer_virtualbox.box which is the same VirtualBox machine but converted
to Vagrants box format. You can now reuse and distribute these files and the users
will upon booting them be able to go to <http://localhost:9090> and see the TYPO3
introduction package in all its glory.

**Configuration details**

Port forwarding is used for SSH, MySQL and HTTP.

**SSH**

  * System user: root
  * System password: autoinstall
  * Host port: 3213
  * Usage (on host): ssh root@localhost -p 3213

**MySQL**

  * Empty password for root user
  * TYPO3 database name: typo3
  * TYPO3 database user: typo3
  * TYPO3 database password: typo3
  * Host port: 3307

**TYPO3**

  * Admin user: admin
  * Admin password: joh316
  * Install tool password: joh316
  * Host port: 9090
  * Usage (on host): <http://localhost:9090> and <http://localhost:9090/typo3>

The nice part is that you now have a base Vagrant box that is pre-installed with a
LAMP stack and TYPO3. You can now create a [Vagrantfile](http://docs.vagrantup.com/v2/vagrantfile/index.html)
to further automate and streamline your TYPO3 development.

I will write a follow-up on using Vagrant with this base box for TYPO3 development
and some insight in automating TYPO3 deployment of configuration and extensions.

### About security

This setup is (on purpose for simplicity) insecure. The root password is readable in the kickstart
file and there is only a root user and no firewall.
For local development this is okay, but obviously not for production/online usage.

I will detail this in the follow-up post once I have used Packer to build production images.
