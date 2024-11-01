---
title: "Compiling Apache 2.4 on Ubuntu 12.04"
date: "2012-11-06"
categories: 
  - "devops-sysadmin"
tags: 
  - "12-04"
  - "2-4"
  - "apache"
  - "compile"
  - "ubuntu"
---

I've decided that I need to up my game when it comes to webservers. However I'm not yet ready to switch to Nginx or one of the other webservers out in the wild as I need something up and running rapidly.

Granted the numbers are definitely against Apache in a lot of benchmarks but historically I've always had a good experience and the entry level makes it much more appropriate for me to stick with it.

However Apache 2.2 is rather long in the tooth, thankfully 2.4 has been out for a while now. The problem I have is that I tend to favour Ubuntu as a platform and there is no sign of a 2.4 version appearing on the horizon anytime soon as they are waiting for it to be implemented upsteam in Debian before including it in Ubuntu.

Now there are PPAs available out there but im not overly happy using them (especially on production environments) So the only option is to compile. <!--more--> First thing is to install all the dependencies we are going to need. Thankfuly ubuntu has a nice and simple way of handling this.

```
apt-get build-dep apache2
```

We can then download the source code and start the compilation.

So from the root of our new copy of the source we need to run our configure.

```
./configure --prefix=/usr/local/apache2 \
 --enable-mods-shared=all \
 --enable-http \
 --enable-deflate \
 --enable-expires \
 --enable-slotmem-shm \
 --enable-headers \
 --enable-rewrite \
 --enable-proxy \
 --enable-proxy-balancer \
 --enable-proxy-http \
 --enable-proxy-fcgi \
 --enable-mime-magic \
 --enable-log-debug \
 --with-mpm=event

```

You will notice that I'm installing it using the event mpm. Hopefully I'll be covering more about the event mpm in the future.

Next we need to run make

```
make && make install
```

Once that's complete you should be able to run

```
/usr/local/apache2/bin/apachectl start
```

and get the "it works" message through your webrowser when accessing the server IP.

Dont forget to configure apache to suit your specific requirements.

Something that will come up is how to start apache on boot. Seeing as Ubuntu uses Upstart it makes sense to utilise it for controlling apache.

So in the file \`/etc/ini/apache.conf\` we need to put

```
# apache2 - http server
#
# Apache is a web server that responds to HTTP and HTTPS requests.
# Required-Start:    $local_fs $remote_fs $network $syslog
# Required-Stop:     $local_fs $remote_fs $network $syslog

author "Matt Cockayne <matt@zucchi.co.uk"
description "Apache 2.4 HTTP Server"

start on runlevel [2345]
stop on runlevel [!2345]

console output

pre-start script
    mkdir -p /var/run/apache2 || true
    install -d -o www-data /var/lock/apache2 || true
    # ssl_scache shouldn't be here if we're just starting up.
    # (this is bad if there are several apache2 instances running)
    rm -f /var/run/apache2/*ssl_scache* || true
end script

# Give up if restart occurs 10 times in 30 seconds.
respawn limit 10 30
respawn

script
    if test -f /usr/local/apache2/bin/envvars; then
        . /usr/local/apache2/bin/envvars
    fi
    ULIMIT_MAX_FILES="ulimit -S -n `ulimit -H -n`"
    if [ "x$ULIMIT_MAX_FILES" != "x" ] ; then
        $ULIMIT_MAX_FILES
    fi

    /usr/local/apache2/bin/httpd -k start -D FOREGROUND
end script

```

This is a rather simple upstart script and I will be looking to update it at some point... but it works

Once that's done you should find that on reboot Apache will start and take advantage of all the management features of upstart including attempting to respawn Apache should it end unexpectedly. You should also be able to then use the following commands to control Apache.

```
# how to start start apache
start apache
# or 
initctl start apache

# how to stop apache
stop apache
# or 
initctl stop apache

# how to restart apache
restart apache 
# or 
initctl restart apache

# check the status of apache 
status apache
# or
initctl status apache
```

I generally tend to avoid using the apachectl script found at /usr/local/apache/bin/apachectl once upstart takes control.
