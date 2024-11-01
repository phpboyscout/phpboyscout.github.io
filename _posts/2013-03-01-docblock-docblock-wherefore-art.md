---
title: "Docblock, Oh Docblock, wherefore art thou Docblock (hint: Zend Optimizer Plus lost them)"
date: "2013-03-01"
categories: 
  - "php"
tags: 
  - "5-4"
  - "apc"
  - "docblock"
  - "eaccellerator"
  - "optimizer"
  - "php"
  - "plus"
  - "reflection"
  - "zend"
---

tl;dr> I make a terrible assumption about Zend Optimizer+ and am corrected by Dominic in the comments;

Terrible post title I know but its the best I could come up with.

I've just come up for air after spending the majority of the day debugging some issues on our current development sandbox.

Now our sandbox tends to be quite bleeding edge in some circumstances and as such we run a fair few bits of unstable code. On the sandbox in question we have been running PHP 5.4.11 and unfortunately we have struggled to get APC working with it just the way we need it to. The lack of APC tends to make this sandbox quite slow.

We recently saw that Zend have open-sourced their OptimizerPlus extension ([https://github.com/zend-dev/ZendOptimizerPlus](https://github.com/zend-dev/ZendOptimizerPlus "https://github.com/zend-dev/ZendOptimizerPlus")) and that it was compatible with 5.4.... Fantastic, or so we thought.

<!--more--> So I added the new OptimiserPlus to the sandbox and everything was going swimmingly. That was until we had to run one of the utility scripts that we use to rebuild some of our data structures. These scripts make use of different parts of both Zend Framework and Doctrine which tend to rely on some heavy DocBlock annotations.

Now having used both APC and Zend Server knowing that they done affect this kind of functionality I had expected that OptimizerPlus would be fine.... Wrongo. It took me a good few hours of head scratching trying to figure out what had happened.

It turns out that OptimizerPlus suffers from the same flaws that eAccellerator does and strips Docblocks when caching the bytecode. This results in Reflection returning false when you call methods such as \`getDocComment()\`.

All in all its not the end of the world I just disable OptimizerPlus and have to wait till I can get APC working. Not my ideal scenario but I can live with it.

Something that does concern me is that there is currently an RFC that has gone to vote ([https://wiki.php.net/rfc/optimizerplus](https://wiki.php.net/rfc/optimizerplus "https://wiki.php.net/rfc/optimizerplus")) about integrating OptimizerPlus into the PHP 5.5 distribution. While this is great I do worry how many other things may break and will they be picked up and fixed for the 5.5 release.

**Update:** Since writing this post the RFC has finished being voted upon and has been approved. You can expect to see Optimizer Plus appearing bundled with PHP soon.

**Update (15th Mar 13):** Thanks to Dominics' comment I now know that you can tell Optimizer+ to retain your Docblocks by setting your config using

```
zend_optimizerplus.save_comments (default "1")
	If disabled, all PHPDoc comments are dropped from the code to reduce the
	size of the optimized code. Disabling "Doc Comments" may break some
	existing applications and frameworks (e.g. Doctrine, ZF2, PHPUnit)

zend_optimizerplus.load_comments (default "1")
	If disabled, PHPDoc comments are not loaded from SHM, so "Doc Comments"
	may be always stored (save_comments=1), but not loaded by applications
	that don't need them anyway.
```

That'll teach me to write a blog post without investigating more first.
