---
title: "Set up SilverStripe 3.1 using only Git (No Composer)"
date: "2013-07-29"
tags: 
  - "composer"
  - "git"
  - "silverstripe"
  - "silverstripe-3-1"
---

We recently tried to use composer to set up SilverStripe 3.1, but ended up with a dependency nightmare. In order to work around this we decided to make use of Git submodules.

First set up your Git repository and run:

```
git init
```

Next set up a site directory for the code inside your Git repository. Then navigate to [SilverStripe Installer](https://github.com/silverstripe/silverstripe-installer) in your browser and Download a copy. Extract files, and copy contents to site folder. Now we need to add the CMS and Framework. Navigate in a browser to the Git Hub repositories for [CMS](https://github.com/silverstripe/silverstripe-cms) and [Framework.](https://github.com/silverstripe/silverstripe-framework) Now copy the HTTPS clone URL for each project and run the following, to add these as Git sub modules.

```
git submodule add https://github.com/silverstripe/silverstripe-framework.git site/framework
git submodule add https://github.com/silverstripe/silverstripe-cms.git <path-to-site>site/cms
```

Now delete mysite/\_config.php and load the site. Follow the normal install instructions displayed and you will have a running version of [SilverStripe 3.1](http://www.silverstripe.org/ "SilverStripe")
