---
layout: post
title: "Mailserver with Postfix, Dovecot and Horde"
description: "How to set up a mailserver with Postfix, Dovecot, Spamassassin, Amavis and Horde"
category: partheas
tags: [tech, hosting, howto]
---
{% include JB/setup %}

Two weeks ago our mail server hosted on our dedicated line at the office went down.

![Image courtesy of http://www.lolcats.com/popular/12041-oh-no-she-di-int.html](/assets/images/lc-oh-no.jpg)

She did.

It was planned to move it outside the office since our dedicated line didn't
proof to be that reliable anyway. Unfortunately, you know how these things go,
it broke before moving it out.

## Our previous setup

We were running Kolab as our groupware solution, which worked pretty well until
I hit 'yum update' a certain day. Yum updated db4 while Cyrus was still running,
which led to a crashed database. Despite several recovery methods and trying setting
back a backup of mailboxes.db it didn't work out.

## Moving away from Cyrus

After spending several hours trying to recover I gave up and decided to move from
Cyrus to Dovecot. In large part because I read multiple stories about those
database issues and the fact that Dovecot uses plain old, standard Maildir to
store its mails which gave me some assurance that a next disaster would be much
easier to recover from.

### Moving our Cyrus mailboxes to Dovecot

There are some cyrus2dovecot scripts out there, but most were for previous versions
of Cyrus. Finally I found this one at the blog of [Gosign media](http://blog.gosign.de/2012/11/11/cyrus-to-dovecot-converting-imap-mailboxes)

This script was nearly perfect. All I had to do was add an extra modifier %n that
returns the username dotted style (instead of the ^ as used by Cyrus) to convert
to our future Dovecot setup. You can find the modified version [here](/assets/files/cyrus2dovecot).

For reference, this is how I called the script:

    ./cyrus2dovecot --cyrus-inbox /root/backup/domain/%g/%d/%h/user/%U  \
      --cyrus-seen /root/backup/domain/%g/%d/user/%h/%U.seen \
      --cyrus-sub /root/backup/domain/%g/%d/user/%h/%U.sub \
      --cyrus-quota /root/backup/domain/%g/%d/quota/%h/user.%U \
      --dovecot-inbox /tmp/dovecot/%U -F 3 -O 1 fn.ln@partheas.com -u

## Our new setup, step by step

The goal of our new setup is twofold. We do not only want to host our own e-mails,
but co-host it with our customers. For us this means that users need to authenticate
on both LDAP (our staff) as well as via a database (our customers).
Luckily with Dovecot this is fairly simple.

What follows is a step-by-step guide on installing and configuring Postfix, Postgresql,
Postfixadmin, Dovecot, Spamassassin and ClamAV. In a second post, we'll install
and configure Horde on a separate machine. And in a final post we'll install and
configure a backup mailserver.

If you're goal is different then please check out the original post of this on
Ex Ratione's blog: <http://www.exratione.com/2012/05/a-mailserver-on-ubuntu-1204-postfix-dovecot-mysql/>
His excellent blogpost covers everything you will need to have a full blown mailserver
for either you or your customers. What I describe in this post is a mere adaptation
for different needs:

* Use Postgresql instead of MySQL
* Use LDAP next to SQL authentication
* Lock the server down with ufw
    * Only allow access to Postfixadmin from company networks
    * Only allow SSH from company networks
    * Block all traffix except the necessary services
* Allow company servers to send mails without authenticating
* (Part 2) Put Horde on a separate machine
    * (Part 2) Plus describe how to setup Horde and use ActiveSync

* (Part 3) Add a backup mailserver

... Update on its way ...
