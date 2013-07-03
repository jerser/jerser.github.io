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

Why [Packer](http://packer.io) instead of Vagrant? Because it supports much more.
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

Where possible, the provisioning/template does not use versions but uses the latest
version, the goal is to automatically rebuild when a new version of TYPO3 or one of
its dependencies is released.

#### Kickstart

The installation is [kickstarted](https://www.centos.org/docs/5/html/Installation_Guide-en-US/s1-kickstart2-file.html)
by this [Anaconda kickstart file]() served by Packers built-in webserver.

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

I just used the inline shell commands since it is still rather limited what has to be done.
I will probably rework it into a shell script though to avoid having to use a webserver
to run my [CasperJS script](https://gist.github.com/jerser/de36a686f3591d4f91d8).
I initially thought/hoped that I could download files from the built-in webserver
from Packer, just like during kickstart ([on its way](https://github.com/mitchellh/packer/issues/118)).

I used a provisioning override to only install the Virtual Box Guest Additions when
the builder is virtualbox.

{% highlight json %}
"override": {
      "virtualbox": {
        "type": "shell",
        "inline": [
          "yum -y install gcc kernel-devel kernel-headers make bzip2",
          "rpm -Uvh http://download.fedoraproject.org/pub/epel/6/i386/epel-release-6-8.noarch.rpm",
          "yum -y install dkms",
          "export KERN_DIR=/usr/src/kernels/`uname -r`",
          "mkdir /mnt/VBoxGuestAdditions",
          "mount -o loop VBoxGuestAdditions.iso /mnt/VBoxGuestAdditions",
          "/mnt/VBoxGuestAdditions/VBoxLinuxAdditions.run"
        ]
      }
    }
{% endhighlight %}

### Conclusion

You can get a zip of the files [here](https://github.com/jerser/packer-templates/archive/master.zip)
or check it on [GitHub](https://github.com/jerser/packer-templates).

#### How to use it?

    packer build typo3/typo3.json

    ==> Builds finished. The artifacts of successful builds are:
    --> virtualbox-typo3: VM files in directory: output-virtualbox-typo3
    --> virtualbox-typo3: 'virtualbox' provider box: packer_virtualbox.box

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

### Security Warning

This setup is (on purpose for simplicity) insecure. The root password is readable in the kickstart
file and there is only a root user and no firewall.
For local development this is okay, but obviously not for production/online usage.

I will later update it to reset all passwords automatically when deployed for
production usage. Keep an eye on this page if you want to know how.
