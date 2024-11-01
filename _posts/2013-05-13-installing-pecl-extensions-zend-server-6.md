---
title: "Installing PECL extensions for Zend Server 6"
date: "2013-05-13"
tags: 
  - "pecl"
  - "phpize"
  - "serverubuntu"
  - "zend"
---

Recently we have revisited using Zend Server for some of our projects and decided to give the new version 6 a chance to prove itself.

Overall its a big improvement over version 5. There are still some things that are extremely annoying but we have decided that we can overlook them.

However there is one thing that we couldn't do without. By default you will find that a number of PECL extensions will not install out of the box (at least this is what we experience using the Debian based install).

To fix this you will need to make sure you install the additional packages in ubuntu

- **php-5.4-source-zend-server** or **php-5.3-source-zend-server** depending on the php version you are using
- **autoconf**
- **build-essential**

Once this is done you should now be able to install extensions from PECL without too much hassle.
