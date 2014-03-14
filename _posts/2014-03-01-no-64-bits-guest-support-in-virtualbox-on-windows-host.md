---
layout: post
title: "No 64 bit guest support on Windows host"
description: ""
tags:
  - windows
  - virtualbox
---
{% include JB/setup %}

If you are running Windows 8(.1) 64 bits and yet you have no 64 bits OS options
in VirtualBox it is likely because Hyper-V is running. Follow the steps below to
disable Hyper-V in Windows 8.

Open 'Turn Windows features on or off'.
!['Turn Windows features on or off' screenshot](/assets/images/turnwindowsfeaturesonoroff.png)

And uncheck 'Hyper-V'
![Checked Hyper-V option screenshot](/assets/images/hypervchecked.png)
