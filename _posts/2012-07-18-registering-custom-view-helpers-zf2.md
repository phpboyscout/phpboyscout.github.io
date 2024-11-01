---
title: "Registering custom view helpers in ZF2"
date: "2012-07-18"
categories: 
  - "php"
---

If you want to register custom view helpers with a module you can do so by using the service location built into the Skeleton Application and creating a module config that looks something like.

```
return array(
    'view_helpers' => array(
        'invokables' => array(
            // generic view helpers
            'truncate' => 'Zucchi\View\Helper\Truncate',

            // form based view helpers
            'bootstrapForm' => 'Zucchi\Form\View\Helper\BootstrapForm',
            'bootstrapRow' => 'Zucchi\Form\View\Helper\BootstrapRow',
            'bootstrapCollection' => 'Zucchi\Form\View\Helper\BootstrapCollection',
        ),
    ),
);

```
