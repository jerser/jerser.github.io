---
layout: post
title: "Using Ansible to configure your Ruby Application Servers and deploy your Ruby Gems"
description: "Use Ansible to configure your Ruby server. Includes Playbooks to deploy your Ruby applications and monitor them with Monit"
tags:
  - ruby
  - rails
  - monitoring
  - automation
---
{% include JB/setup %}

**Situation**: You have a Ruby (not Rails) software application which is meant to run
as a daemon and you have to deploy this application to multiple servers at once.
You want to do it right, so you don't want to login on the server and manually start
installing things. Great! You've come to the right place, in this post I will step by
step describe how to get from your fresh server image to a fully functional Ruby
server, monitoring your Ruby process(es) included.

### Goal

I have multiple Ruby projects which I want to install on multiple remote servers.
I want these Ruby projects to run as daemons and I want to monitor these processes,
the use case here will be to restart the process if it crashes and start the process
on boot.

#### What I'll show

We'll install Ansible on your machine. Create some Playbooks to install all
necessary software on the remote server. Configure [Monit](http://mmonit.com/monit/)
to monitor our Ruby processes. Create a separate single Playbook to copy your locally
built Gem files to the remote server, install them and have them run as a service.

### Prerequisites

First off, you will need access to a server with Ubuntu installed. If you use another
distro, the main thing you'll have to change is the use of [apt](http://docs.ansible.com/apt_module.html)
in the Ansible playbooks and the package names there.

Secondly, in my case I built my Ruby projects as Gems. I will not distribute them
through [Rubygems](https://rubygems.org) nor [Gemfury](https://gemfury.com), but will
just build them locally and copy them to the server for local installation. The Gem
used in the example has an [executable](http://guides.rubygems.org/specification-reference/#executables) named myapp that is used to start and stop
the process as a daemon (using [Dante](https://github.com/nesquena/dante)).

### Quick introduction into Ansible and its concepts

Ansible is an automation tool, it allows you to write [Playbooks](http://docs.ansible.com/playbooks.html),
Playbooks describe (in a YAML format) a list of tasks to be executed on a set of hosts.

Example playbook:
{% highlight yaml %}
---
- hosts: all
  remote_user: root
  tasks:
  - name: Ensure Apts package cache is up to date
    shell: apt-get update
  - name: Ensure python-apt is present
    command: apt-get install python-apt
{% endhighlight %}

So in the above example you see the hosts declaration, this allows to define which
set of tasks is for which type of host. You could have a webserver hosts group,
an application server hosts group, etc. In this example I specified all which simply
means that the tasks following it apply for *all* hosts. Ansible uses a special file,
named [inventory](http://docs.ansible.com/intro_inventory.html) in Ansible lingo
to specify which hosts label applies to which physical hostname or IP address.

Example inventory file with no hosts groups, but only an IP address

```
10.0.3.2
```

Another example, this time with hosts groups.

```
[web]
10.0.3.2

[app]
10.0.3.1
10.0.3.3
```

The inventory is pretty flexible, you can read it from file, but even read it from
remote services, check the [documentation](http://docs.ansible.com/intro_inventory.html)
for more information.

Back to the previous Playbook example:
{% highlight yaml %}
---
- hosts: all
  remote_user: root
  tasks:
  - name: Ensure Apts package cache is up to date
    command: apt-get -y update
  - name: Ensure python-apt is present
    command: apt-get -y install python-apt
{% endhighlight %}

remote_user is the user Ansible will use to logon on the remote machine, and the user
used to execute the tasks. You can use a non-root user and use sudo when needed as well.

Tasks is a list of tasks, every task has a name (used for visual feedback when executing them)
and a module name. After the module name you configure what it should do. Ansible
ships with a bunch of [modules](http://docs.ansible.com/list_of_all_modules.html).
In the example we us the the [command](http://docs.ansible.com/command_module.html) module.
This module allows to execute one specific command on the remote node.

If you already looked at the list of available modules you'll see that there is an
apt module to install packages. For that to work you need python-apt though, so that
is why for the first command I simply install this package. All next playbooks can then
use the apt module.

Now when creating these Playbooks try to keep in mind that you want to be able
to re-run these playbooks often. So write your tasks in a fashion that they can
be re-run without destroying your current set-up. So don't write it in a way that
initially installs everything, write them as such that they update everything:
The first time this would mean install everything, the second time
this would mean update everything to the latest versions or changes.

### Getting started

#### Install Ansible locally

One of the main reasons why I prefer Ansible over other tools like Chef is the fact
that it doesn't use any agents on the server, so as long as Python 2.5+ is installed
(it is by default on most if not all distributions) on the remote nodes you don't
need to do anything on the remote node to get started.

I installed Ansible locally through Pip, but they have native packages available.
Check [this page](http://docs.ansible.com/intro_installation.html#installing-the-control-machine).

```
$ sudo pip install ansible
```

#### Copy your SSH public key to the remote node(s)

```
$ ssh-copy-id root@<your remote IP address>
```

#### Create a directory to store your Playbooks


```
$ mkdir -p ~/Ansible/Playbooks
```

#### Create your inventory file

In this example I will only add one host to the inventory file and I will not
use the default location of the inventory file, contrary to
[Ansibles documentation](http://docs.ansible.com/intro_inventory.html).
I prefer to store this on a per project basis.

```
$ echo "10.0.2.3" > ~/Ansible/inventory
```

#### Creating the Playbooks

The following Playbooks will install/update an Ansible dependency (python-apt), will
install/update cURL and Monit.
It will also download and build Ruby 2.1.2.

```
$ vim ~/Ansible/Playbooks/init.yml
```
{% highlight yaml %}
---
- hosts: all
  remote_user: root
  tasks:
  - name: Ensure Apts package cache is up to date
    shell: apt-get update
  - name: Ensure python-apt is present
    command: apt-get install python-apt
{% endhighlight %}

```
$ vim ~/Ansible/Playbooks/system.yml
```
{% highlight yaml %}
---
- hosts: all
  remote_user: root
  tasks:
  - name: Ensure cURL is present and up to date
    apt: pkg=curl state=latest
  - name: Ensure Monit is present and up to date
    apt: pkg=monit state=latest
{% endhighlight %}

```
$ vim ~/Ansible/Playbooks/ruby.yml
```
{% highlight yaml %}
---
- hosts: all
  remote_user: root
  tasks:
  - name: "Ensure Ruby prerequisite: build-essential is latest version"
    apt: pkg=build-essential state=latest
  - name: "Ensure Ruby prerequisite: zlib1g-dev is latest version"
    apt: pkg=zlib1g-dev state=latest
  - name: "Ensure Ruby prerequisite: libyaml-dev is latest version"
    apt: pkg=libyaml-dev state=latest
  - name: "Ensure Ruby prerequisite: libssl-dev is latest version"
    apt: pkg=libssl-dev state=latest
  - name: "Ensure Ruby prerequisite: libgdbm-dev is latest version"
    apt: pkg=libgdbm-dev state=latest
  - name: "Ensure Ruby prerequisite: libreadline-dev is latest version"
    apt: pkg=libreadline-dev state=latest
  - name: "Ensure Ruby prerequisite: libncurses5-dev is latest version"
    apt: pkg=libncurses5-dev state=latest
  - name: "Ensure Ruby prerequisite: libffi-dev is latest version"
    apt: pkg=libffi-dev state=latest
  - name: "Download Ruby 2.1.2"
    get_url: url=http://cache.ruby-lang.org/pub/ruby/2.1/ruby-2.1.2.tar.gz sha256sum=f22a6447811a81f3c808d1c2a5ce3b5f5f0955c68c9a749182feb425589e6635 dest=/root/ruby-2.1.2.tar.gz
    register: download_ruby
  - name: "Unpack Ruby 2.1.2"
    shell: mkdir -p /root/ruby-2.1.2 && tar xzf /root/ruby-2.1.2.tar.gz --strip 1 -C /root/ruby-2.1.2
    when: download_ruby|changed
  - name: "Configure, make and make install Ruby 2.1.2"
    shell: cd /root/ruby-2.1.2 && ./configure && make && make install
    when: download_ruby|changed
  - name: "Remove build artifacts"
    command: rm -rf /root/ruby-2.1.2
    when: download_ruby|changed
{% endhighlight %}

This one installs the source dependencies of Ruby so we can compile it from source.
The last four tasks use something new. The first one, the download task registers the
result in the download_ruby variable. The get_url module registers in this variable
whether it downloaded something or not. This is then used to determine whether or not
we need to build Ruby (again) by using when as part of our following tasks.
This prevents building and installing Ruby every single time the Playbook is run.

Now I create one final YAML file that includes all the previous ones so I can run
everything at once.

```
$ vim ~/Ansible/bootstrap.yml
```
{% highlight yaml %}
---
- include: Playbooks/init.yml
- include: Playbooks/system.yml
- include: Playbooks/ruby.yml
{% endhighlight %}

And to run it:

```
$ ansible-playbook -i ~/Ansible/inventory ~/Ansible/bootstrap.yml
```

After this you're server should be ready to host your Ruby applications.

### Using Ansible to deploy your Gems

I make a separate Playbook that does the actual deploying of the Gem(s). Besides
copying and installing the Gems this playbook will also make sure all dependencies
are met for the native extensions that might be used (in this case Sqlite and Nokogiri).

```
$ vim ~/Ansible/Playbooks/myapp_dependencies.yml
```
{% highlight yaml %}
---
- hosts: all
  remote_user: root
  tasks:
  - name: Ensure Sqlite3 is latest version
    apt: pkg=sqlite3 state=latest
  - name: Ensure libsqlite3-dev is latest version
    apt: pkg=libsqlite3-dev state=latest
  - name: Ensure libxml2-dev is latest version
    apt: pkg=libxml2-dev state=latest
  - name: Ensure libxslt-dev is latest version
    apt: pkg=libxslt-dev state=latest
  - name: Ensure Myapp user exists (myapp)
    user: name=myapp createhome=no system=yes shell=/bin/bash home=/opt/myapp state=present
  - name: Ensure /opt/myapp exists
    shell: mkdir -p /opt/myapp/pids && mkdir -p /opt/myapp/logs && mkdir -p /opt/myapp/config && chown -R myapp.myapp /opt/myapp
{% endhighlight %}

This Playbook will install the dependencies for sqlite3 and nokogiri. It will also create
a system user **myapp** and it will create an application directory at /opt/myapp

```
$ vim ~/Ansible/Playbooks/myapp_deploy.yml
```
{% highlight yaml %}
---
- hosts: all
  remote_user: root
  tasks:
  - name: Recreate deployment directory
    shell: "rm -rf /root/deploymyapp && mkdir /root/deploymyapp"
  - name: Copy Gem to server
    copy: src=~/MyApp/.out/ dest=/root/deploymyapp
  - name: Stop application
    monit: name=myapp state=stopped
  - name: Stop Monit
    service: name=monit state=stopped
  - name: Ensure current version of Gem is removed
    gem: name=myapp state=absent
  - name: Install new version of MyApp Gem
    gem: name=myapp gem_source=/root/deploymyapp/myapp.gem user_install=no state=present
  - name: Copy Monit scripts
    copy: src=~/MyApp/monit/ dest=/etc/monit/conf.d/
  - name: Start Monit
    service: name=monit state=started
  - name: Start application
    monit: name=myapp state=started
  - name: Copy logrotate configuration
    copy: src=~/MyApp/config/logrotate.conf dest=/opt/myapp/config/logrotate.conf mode=444
  - name: Ensure Logrotate Cron Job is present
    cron: name=Logrotate user=myapp hour=6 state=present job="/usr/sbin/logrotate -f -s /opt/myapp/logrotate.status /opt/myapp/config/logrotate.conf"
{% endhighlight %}

```
$ vim ~/Ansible/myapp.yml
```
{% highlight yaml %}
---
- include: Playbooks/myapp_dependencies.yml
- include: Playbooks/myapp_deploy.yml
{% endhighlight %}

This Playbook creates a directory on the remote server, copies Gem files
from my local machine to that remote directory. Stops the currently running application
through Monit, Stops Monit (to prevent restarting the just stopped application),
removes the currently installed Gem, installs the new copy, copies the monit monitoring
script and it starts Monit and the application again.

This is what I typically would have in the Monit script:

```
check process myapp with pidfile /opt/myapp/pids/myapp.pid
start program = "/usr/local/bin/myapp start"
stop program = "/usr/local/bin/myapp stop"
```

This will make sure that Monit monitors the PID file, checks if the PID in that file
is a running process, if not it will execute the command defined in start program.
This also makes it redundant to create a separate upstart script to start our application
on boot. Monit will see that our application is not running when it starts itself
and thus it will start our application.

You will also see that I create a cron job to logrotate the logs of MyApp.
The logrotate.conf file which I copy in the Playbook looks like this. The playbook
installs a logrotate cron job to run every day at 6AM.

```
/opt/myapp/logs/* {
  daily
  copytruncate
  compress
  compresscmd /bin/bzip2
  compressext .bz2
  rotate 31
  start 0
  maxage 100
}
```

#### Using Rake to tie it together

I never call this Playbook directly through Ansible, I typically have something
like below in my projects Rakefile that will build the Gem(s) and call the Ansible
deployment Playbook afterwards. The example given is for a project with multiple
separate Gems, it will of course work just fine for one Gem as well.

```
$ vim ~/MyApp/src/Rakefile
```
{% highlight ruby %}
namespace :gem do
  task :build do
    `rm -rf #{File.dirname(__FILE__)}/.out && mkdir #{File.dirname(__FILE__)}/.out`
    Dir['**/*.gemspec'].each do |gemspec|
      target = File.dirname(gemspec)
      Dir.chdir(target) do
        puts "Building #{File.basename(target, '.rb')} gem"
        `gem build #{File.basename(gemspec)}`
        Dir['*.gem'].each do |gemfile|
          `mv #{gemfile} #{File.dirname(__FILE__)}/.out/#{File.basename(gemspec, '.gemspec')}.gem`
        end
      end
    end
  end

  task :deploy => [:build] do
    IO.popen('ansible-playbook -i ../Ansible/inventory ../Ansible/myapp.yml') do |stdout|
      while line = stdout.gets do
        puts line
      end
    end
    puts
  end
end
{% endhighlight %}

Running **rake gem:deploy** will now build all the projects Gems and deploy them to
the servers in the inventory file.

### That's it!

You now have a setup that will convert a minimal Ubuntu installation into a
Ruby environment. You'll also have a deployment Playbook that will deploy the latest
version of your Gems to all servers and finally your Ruby processes are monitored
for downtime by Monit and we even rotate the logs daily.
This should make your Ruby application ready to run happily in production.

This is based on a setup I use in production, it is however possible that while
converting it to a blog post I missed some pieces, let me [know](mailto:jeroen@serpieters.be)
if that is the case!
