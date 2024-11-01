---
title: "Connecting to Ubuntu 18.04+ using RDP"
date: "2019-05-20"
categories: 
  - "devops-sysadmin"
---

We have a mix of different setups that the Software Engineer and Data Scientists use to get their work done. There are some using just Linux on laptops, Some on MacBooks and some on the various versions of Windows.

For those not using Linux as their primary OS we have a bunch of Desktops that run Ubuntu 18.04+ for them to connect to. SSH can do quite a lot but a few of the team work remotely and in house we prefer RDP for that kind of thing rather than VNC.

We have had some issues with connections in the past so this post exists to remind me how next time I need to set it up. First we need to install the xRDP server package.

```bash
sudo apt install xrdp
```

Next we need to ensure that we have the right ports open on the workstation. If like me you also use UFW to manage your firewall rules then open port 3389 using...

```bash
sudo ufw allow 3389
```

The issue left, is that you will get an annoying pop up when you log in about a colour management profile needing to be set up and asking you to provide your password. Even then you may still get some annoying crash pop-ups.

I found a really good solution to this at [http://c-nergy.be/blog/?p=12043](http://c-nergy.be/blog/?p=12043) which I've cribbed and paraphrased below

Create the file `/etc/polkit-1/localauthority/50-local.d/45-allow-colord.pkla` (using your editor of choice and `sudo`)and add the following contents

```
[Allow Colord all Users]
Identity=unix-user:*
Action=org.freedesktop.color-manager.create-device;org.freedesktop.color-manager.create-profile;org.freedesktop.color-manager.delete-device;org.freedesktop.color-manager.delete-profile;org.freedesktop.color-manager.modify-device;org.freedesktop.color-manager.modify-profile
ResultAny=no
ResultInactive=no
ResultActive=yes

```

We now need to clear any crash dumps from the workstation

```
sudo rm /var/crash/*
```

You should then be good to connect to using whatever RDP client you prefer... I like [Remmina](https://remmina.org/) myself but each to their own.
