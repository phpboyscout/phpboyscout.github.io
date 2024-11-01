---
title: "Disabling Cache in Silverstripe 3.1"
date: "2013-07-31"
tags: 
  - "cache"
  - "silverstripe"
  - "silverstripe-3-1"
---

While working with Silverstripe we found ourselves having to run "?flush=1" a lot to clear the Cache. To switch it off, while you work, add the following to your mysite/\_config.php:

```
SS_Cache::set_cache_lifetime('default', -1, 100);
```
