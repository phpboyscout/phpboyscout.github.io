---
title: "Enabling MYSQL_CLIENT_INTERACTIVE with Doctrine 2 on Rackspace Cloud Database"
date: "2013-07-26"
tags: 
  - "doctrine"
  - "doctrine-2-0"
  - "mysql"
  - "mysqli"
  - "rackspace"
  - "rackspace-cloud"
  - "rackspace-cloud-database"
  - "zucchidoctrine"
---

We recently ran into problem using Doctrine 2 connecting to a Rackspace Cloud Database using the MySqli Driver.

Problem:

We have a long running PHP script that can sometimes run for hours at a time whilst processing information. This script requires a connection to a database, but has long periods of inactivity where there is no actual interaction with MySQL. By default MySQL uses the "wait\_timeout" setting which states, how long an inactive connection can exist before it is killed. This is normally fine with web pages requests, as it is usually a short lived request. Unfortunately you do not have the ability to alter this setting when using Rackspaces Cloud Database.

Solution:

When using the MySQLi extension you can create a connection in "interactive mode" by passing the "MYSQLI\_CLIENT\_INTERACTIVE" flag, which will then use the "interactive\_timeout" setting. On Rackspace this is set to 8 hours!

Annoyingly Doctrine does not allow you to pass any flags to the MySQLi Connection. So we overrode Doctrine\\DBAL\\Driver\\Connection with our own [Driver](https://github.com/zucchi/ZucchiDoctrine/blob/master/src/ZucchiDoctrine/Driver/Mysqli/MysqliConnection.php "ZucchiDoctrine/Driver/Mysqli/MysqliConnection.php") which then allows us to pass a "flags" parameter through.

Feel free to look at some of the other helpful features in we have added to Doctrine 2 here: [ZucchiDoctrine](https://github.com/zucchi/ZucchiDoctrine "ZucchiDoctrine")
