---
title: "Dell DisplayLink D6000 & Ubuntu 18.04+ Issues"
date: "2019-05-14"
tags: 
  - "dell"
  - "ubuntu"
cover-img: "/assets/images/20190514_124153.png"
---

I love Ubuntu... I'm pretty fond of dell kit too!

So I was rather chuffed when I started working at [Medicines Discovery Catapult](https://md.catapult.org.uk) because they let me have both. When you look at my desk it looks like it could be an advert for Dell. Laptop, monitors, dock, keyboard and mouse.... its great when you have a corporate account with a Dell reseller

![](/assets/images/20190514_124153.png)

However while I've had a lot of success with the D3000 DisplayLink dock on Ubuntu I found that I'm now having to deal with the upgraded D6000... which doesn't play very nicely with the more recent versions of Ubuntu (we are talking 18.04 and later)

I kept finding that after a random amount of time the D6000 would randomly seem to power down... I would lose the screens, audio, networking and USB. and the only way I could fix it is to unplug it from teh laptop and plug it back in. Not ideal, especially if I'm in the middle of a video call or debugging something on the net

Being the kind of techie I am my first port of call checking my logs... but I couldn't see anything that would cause this random disconnect. So off to google I went... eventually I found a lot of information telling me it was part of power management causing things to start powering down... In this case it implied that it was something trying to suspend USB... which sounded really plausible!

So a little more research suggested that I should be using laptop mode tools to disable the ability for USB to be suspended. I gave it a go, though I was dubious as in my mind I shouldn't have needed to install an additional package (albeit a great one for tweaking your power management on a laptop running Linux)

Alas no joy! And I had too much to do to start debugging in depth and ripping apart other peoples code to figure it out.

What did I do? you ask. Well, I just put up with it for a few weeks, but gradually it began to grate on my nerves. However there was that one day where it didn't turn off... and that left me perplexed... I checked if any updates had been applied in my last `apt update && apt upgrade` ... nothing.... it then dawned on me that I had plugged in the headset I used for conference calling into the audio in/out on the dock instead of directly into the laptop.

Now I had a little more information I was able to deduce (with googles help) that the laptop was actually suspending USB, but that the trigger was actually pulseaudio. At this point it becomes really easy to solve the problem.

## Solution

Edit `/etc/pulse/default.pa` using your preferred editor (and `sudo`)

Find the line

```bash
### Automatically suspend sinks/sources that become idle for too long
load-module module-suspend-on-idle
```

And comment it out and save!

Lastly, because its run as a user service you need to restart the Pulse Audio daemon using the command

```bash
systemctl --user restart pulseaudio.service
```

or you could just logout and back in again
