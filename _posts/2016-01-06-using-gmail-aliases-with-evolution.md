---
title: "Using Gmail aliases with Evolution"
date: "2016-01-06"
---

If your anything like me you have a large number of email aliases that you use with Gmail which is great. However I use [Evolution](https://wiki.gnome.org/Apps/Evolution) as a mail client more often than not when using [Gnome3](https://www.gnome.org/) as a desktop.

It's very easy to set up Evolution to create separate outbound email accounts that you can use for handling all of your aliases. It doesn't yet support OAuth2 as an authentication mechanism for any account that is not set up using the built-in Gnome Online Accounts integration.

This is a real pain as Google have disabled the more common 'plain' and 'login' authentication mechanisms for use with an SMTP only account. Meaning that any time that you try to connect to smtp.gmail.com:587 with STARTTLS you will get some form of error message to the effect of "Bad Authentication".

Hopefully I'll find a workaround at some point in the near future or Evolution will add the facility to enable OAuth2 as an available authentication mechanism.

In the mean time there is a workaround if you visit [https://www.google.com/settings/security/lesssecureapps](https://www.google.com/settings/security/lesssecureapps) you can enable these less secure authentication mechanisms allowing you to once again connect and send email via email addresses using SMTP
