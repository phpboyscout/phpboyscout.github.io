---
title: "Loaded Testing"
date: "2012-06-30"
categories: 
  - "devops-sysadmin"
tags: 
  - "apache"
  - "java"
  - "jmeter"
  - "linux"
  - "load"
  - "testing"
  - "ubuntu"
---

I recently had to do some load testing for a site recently that would allow me to test in excess of 100k requests in a 60 second period...

[![JMeter](images/jmeter-logo.jpg "jmeter-logo")](http://jmeter.apache.org/)

So I decided to do some testing using JMeter as it seemed like a suitable tool for doing what I needed and I had used it for some simpler testing in the past.

After a little fumbling around I managed to get a test plan designed that would simulate 10k users actually navigating the site and adding to a cart etc, with a number of various interactions. It wasnt perfect but it would correctly simulate over 100k requests.

So feeling quite pleased with myself I started the test from my laptop. Now I'm not a big gamer, I'm known to play a little World or Warcraft from time to time but that's about it. So when it comes to computing power i tend to opt for battery life over sheer grunt.

Suffice to say, my laptop fell flat on its face, and if it hadn't it turns out that the connection I was using just wasn't up to the task of handling that much traffic adequately.

So plan B...

<!--more-->

I quickly fired up the largest AWS instance available and got a copy of jmeter installed. A little tinkering with my test plan and some googling on how to run jmeter without a gui and a quick

`./jmeter -n -t test-plan.jmx`

and it appeared to be running.

(Please bear in mind that I'm being overly kind... it took a LOT of tinkering and twice as much Googling to work out how to get the test results out so i could actually get some idea of WTF was happening during the test)

So... client "happy"... I decided to go and find a better way to do my load testing in the future.

Sticking with JMeter I managed to find this gem of a page

[http://jmeter.apache.org/usermanual/remote-test.html](http://jmeter.apache.org/usermanual/remote-test.html "Remote Testing")

tl;dr > use your local install of jmeter to trigger tests to run on one or more remote "nodes" and then have all the results sent to your local install.

So I set to work!

## **Building a Node**

First I need to set up an AWS instance that we can use and duplicate so I can quickly build a cluster of nodes on demand. I'm a big fan of Ubuntu so I spin up a micro instance of 12.04 server. Next I shell into the instance and install the default Java runtime from apt

`apt-get install openjdk-7-jre`

Yes I know there are other more appropriate runtimes, but i dont really care... i just need it to work and it does.

next I grab a copy of the latest stable from [http://jmeter.apache.org/download\_jmeter.cgi](http://jmeter.apache.org/download_jmeter.cgi "Download Apache JMeter") and un-tar it to `/usr/local/jmeter`

(N.B. JMeter is available through the apt but I had issues with that version and you need to make sure that both your local version and all the nodes run the same version of jmeter)

We can now test that the install is working running `/usr/local/jmeter/bin/jmeter-server` and you should get some output that looks similar to

```
Created remote object: UnicastServerRef [liveRef: [endpoint:[10.???.???.???:38939](local),objID:[46522b57:138381f1023:-7fff, 2635011707874933136]]]
```

Which tells us that the server is running.

**BUT** unfortunately its not going to work just yet. Because we are using Amazons EC2 we are going to relying on their NAT for routing. Out of the box JMeter just wont work properly.

However there is something we can do to combat this. We can set the parameter `RMI_HOST_DEF` that the `/usr/local/jmeter/bin/jmeter-server` script will include in starting the server.

```
export RMI_HOST_DEF=-Djava.rmi.server.hostname=$(wget http://169.254.169.254/latest/meta-data/public-hostname -q -O -)
```

I'll explain what we are doing here. Amazon have been quite clever by providing a [meta-data endpoint](http://docs.amazonwebservices.com/AWSEC2/latest/UserGuide/AESDG-chapter-instancedata.html) that you can poll from within your instance to get key pieces of data... Including the public dns record.

We can use this endpoint and using wget pipe that into the `RMI_HOST_DEF` param (ensuring that we prepend `-D`) and then export that so it becomes available to the `/usr/local/jmeter/bin/jmeter-server` script.

Now to get the server to start on boot.

a quick upstart script should solve this

```
# Upstart script to initialise jmeter-server

description "JMeter Server"
author "Dev in Charge "

start on started networking
stop on stopping networking
stop on stopping shutdown

console output

script
    # get the current public DNS record
    export RMI_HOST_DEF=-Djava.rmi.server.hostname=$(wget http://169.254.169.254/latest/meta-data/public-hostname -q -O -)

    # start jmeter in server mde
    /usr/local/jmeter/bin/jmeter-server
end script
```

saving this to `/etc/init/jmeter-server.conf` will mean that it will auto-start jmeter-server on boot and allow you to manually control the process using `start jmeter-server` and `stop jmeter-server`

and thats it... instance configured

[![Powered by AWS](images/AWS_Logo_PoweredBy_300px.png "AWS_Logo_PoweredBy_300px")](http://aws.amazon.com/)All you need to do now is save the instance as an AMI and you have an on-demand image for spinning up a cluster of remote JMeter servers for you to play with.

## Configuring your local installation

Now that the server side is working we need to configure our local installation to allow it to connect.

First things first however, make sure you are using the same version of JMeter as you are running on the server.

We need to edit the `jmeter.properties` file that can be found in the bin folder of the installtion you downloaded. Look for the parameter `remote_hosts` This needs to be set with the public dns of the remote server(s) your connecting to. for example

```
remote_hosts=ec2-176-34-164-170.eu-west-1.compute.amazonaws.com,ec2-123-34-456-789.eu-west-1.compute.amazonaws.com
```

Thats your local version configured. You will now be able to tell your local version to run tests on any or all of your specified remotes.

However if your like me you work behind a router/firewall. If so this isnt the end of the story. When you send a test plan to a remote from your local install it will also send the IP address of your local machine for it to send the results back to. JMeter does this by looking up where your current hostname resolves to. In my circumstance it resolved to `127.0.1.1`. The reason it did this is down to the fact my systems host file had the line

```
127.0.1.1    devincharge.local
```

To resolve this I had to change it to my external IP address

```
89.345.871.79    devincharge.local
```

And set up port forwarding from my router to my local machine for all ports from 1024 to 65535. Now, you can if you want use specific ports so you dont have to port forward everything from your router, but i'll leave that for you to lookup as there are plenty resources on how to do this for you to google and I've waffled on for far too long already.

Happy testing
