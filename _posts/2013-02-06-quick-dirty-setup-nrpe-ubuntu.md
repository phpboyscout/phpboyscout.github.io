---
title: "Quick and easy setup of and connection to NRPE on Ubuntu"
date: "2013-02-06"
categories: 
  - "python"
tags: 
  - "icinga"
  - "monitoring"
  - "nagios"
  - "nrpe"
  - "remote"
  - "ubuntu"
---

## About NRPE

NRPE (Nagios Remote Plugin Executor) is a useful tool that allows you to execute scripts on remote servers and return the output for ingestion by some form of monitoring software.

## Setup

We currently have our own instance of Icinga running to monitor our servers and have recently started to offer access to it for our clients.

The majority of our servers (and our clients servers if we set them up) use one variant or another of Ubuntu. This means we can very quickly get our servers connected to a Nagios/Icinga instance.

First things first we need to install the nrpe server and all the associated plugins

```
apt-get install nagios-nrpe-server \
nagios-plugins-basic \
nagios-plugins \
nagios-plugins-extra
```

<!--more-->

Next we need to edit the main nrpe config file to be found @ /etc/nagios/nrpe.cfg. What your looking for is the lines

```
# ALLOWED HOST ADDRESSES
# This is an optional comma-delimited list of IP address or hostnames 
# that are allowed to talk to the NRPE daemon.
#
# Note: The daemon only does rudimentary checking of the client's IP
# address.  I would highly recommend adding entries in your /etc/hosts.allow
# file to allow only the specified host to connect to the port
# you are running this daemon on.
#
# NOTE: This option is ignored if NRPE is running under either inetd or xinetd

allowed_hosts=127.0.0.1

# COMMAND ARGUMENT PROCESSING
# This option determines whether or not the NRPE daemon will allow clients
# to specify arguments to commands that are executed.  This option only works
# if the daemon was configured with the --enable-command-args configure script
# option.  
#
# *** ENABLING THIS OPTION IS A SECURITY RISK! *** 
# Read the SECURITY file for information on some of the security implications
# of enabling this variable.
#
# Values: 0=do not allow arguments, 1=allow command arguments

dont_blame_nrpe=0
```

You will want to change this to the IP of your Nagios/Icinga instance and set the dont\_blame\_nrpe value to 1. Feel free to take a look round the rest of the file. Its all quite interesting and generally will documented. Be careful what you change though in case something breaks.

You will also want to look for some lines that are refererd to as "COMMAND DEFINITIONS" and look something like this

```
command[check_users]=/usr/lib/nagios/plugins/check_users -w 5 -c 10
command[check_load]=/usr/lib/nagios/plugins/check_load -w 15,10,5 -c 30,25,20
command[check_hda1]=/usr/lib/nagios/plugins/check_disk -w 20% -c 10% -p /dev/hda1
command[check_zombie_procs]=/usr/lib/nagios/plugins/check_procs -w 5 -c 10 -s Z
command[check_total_procs]=/usr/lib/nagios/plugins/check_procs -w 150 -c 200
```

You can go ahead and comment these out as we will be adding our own definitions shortly. The main reason for removing these is that we will be configuring some specific scripts for our own use later that allow you to configure your requirements and thereshold from within your Nagios/Icinga config.

## Configuration of Monitoring Server

Once this is complete you can now configure a new "check command" for use with your nagios/icinga server.

```
define command {
                command_name                          check_nrpe
                command_line                          $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$
}

define command {
                command_name                          check_nrpe_command_args
                command_line                          $USER1$/check_nrpe -H $HOSTADDRESS$ -c $ARG1$ -a $ARG2$
}
```

Here you can see that we have set up 2 different check commands. The first is a simple command requiring only one argument of $ARG1$ which would be the name of the command we want to run on the remote server. The second command is almost identical except for the fact it takes a second argument which allows you to input a series of "arguments" to be passed to the command on your remote server. each argument should be separated by a space.

Now that you have these you can then configure your hosts and services to make use of it. I would recommend having a trawl through the Nagios/Icinga sites & documentation to find out how to create a config that suits you.

## Configuration of Remote Server

Now that we have our monitoring server ready its time to add the command we want to run to the remote server.

To do this your /etc/nagios/nrpe.cfg shoudl hopefully have a line in it that looks like

```
include=/etc/nagios/nrpe_local.cfg
```

if it doesn't have a line like that then add it and edit the \`/etc/nagios/nrpe\_local.cfg\` file to look a little like this

```
command[check_apt]=/usr/lib/nagios/plugins/check_apt
command[check_users]=/usr/lib/nagios/plugins/check_users -w $ARG1$ -c $ARG2$
command[check_load]=/usr/lib/nagios/plugins/check_load -w $ARG1$ -c $ARG2$
command[check_disk]=/usr/lib/nagios/plugins/check_disk -w $ARG1$ -c $ARG2$ -p /dev/sda1
command[check_procs]=/usr/lib/nagios/plugins/check_procs -w $ARG1$ -c $ARG2$ -s $ARG3$
command[check_zombie_procs]=/usr/lib/nagios/plugins/check_procs -w 5 -c 10 -s Z
command[check_total_procs]=/usr/lib/nagios/plugins/check_procs -w 150 -c 200 
```

These are a few simple commands that I tend to use most often. These translate to your "check\_nrpe" commands like so

- $ARG1$ = everything inside the square brackets \[ \]
- $ARG2$ = each of the $ARG?$ keys as a single string separated by a space

Once that's done you should be able restart your nrpe server with \`/etc/init.d/nagios-nrpe-server restart\`

It really is that simple. Do bear in mind that because you can pass arbitrary arguments into nrpe this was you could leave yourself vulnerable to a bit of maliciousness so its a good idea to make sure your firewall restricts port 5666 (the default port) to IPs you trust.
