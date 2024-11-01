---
title: "Creating Custom Routes in Silverstripe 3.1"
date: "2013-07-31"
tags: 
  - "routes"
  - "silverstripe"
  - "silverstripe-3-1"
---

We wanted to create a Route to our custom Products Controller in our products module for SilverStripe 3.1, such as: "http://www.examplesite.com/products/<product-slug>"

However looking at the [Controller Documentation](http://doc.silverstripe.org/framework/en/3.1/topics/controller "Controller Documentation") it was not clear how to create a route without an Action being supplied. In our example above the action is not specified, as we just want to use 'view'.

<!--more-->

Solution:

Create a <module-name>/\_config/routes.yml file containing the following:

```
---
Name: productsroutes
After: 'framework/routes#coreroutes'
---
Director:
  rules:
    'product': 'Product_Controller'
---
```

The above will redirect any Url that starts with "/product" to our Product\_Controller. Note that everything after the rule, so after "/product", is used in the next bit for matching.

Now we need to add `private static $url_handers` to Product\_Controller to match our path, so in this example we need to match "$Slug!" which will match "<product-slug>". Note the ! means the slug is required. Of course we want to direct this to a specific action, in this case "view", this gives us:

```
private static $url_handlers = array(
    '$Slug!' => 'view',
);
```

Now just add "view" to the $allow\_actions and add the "view" function. This gives the final Product\_Controller as follows:

```
class Product_Controller extends Page_Controller
{
    private static $url_handlers = array(
        '$Slug!' => 'view',
    );

    private static $allowed_actions = array('view');

    public function view(SS_HTTPRequest $request)
    {
        // Your action code goes here
        return $this->render();
    }
}
```

Handy note:

You can put ?`debug_request=1 on the end of your URL to see how it determines which Controller to use.`
