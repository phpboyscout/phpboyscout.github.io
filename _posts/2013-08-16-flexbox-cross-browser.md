---
title: "Flexbox cross browser"
date: "2013-08-16"
tags: 
  - "chrome"
  - "css"
  - "css3"
  - "firefox"
  - "flexbox"
  - "flexbox-legacy"
  - "ie"
---

Despite having been around for a while and having been through a couple of revisions, its support across browsers can vary greatly. From "Candidate Recommendation" on Chrome/Opera, "legacy flexbox" on Firefox and no support at all on IE9 and earlier.

Making flexbox work consistently across browsers was a challenge for us on a recent project, but I have found a solution that seems to work quite well. <!--more-->

Below is an SCSS @mixin that will attempt to handle compatibility between CR and legacy cross browsers flexbox.

```
@mixin flex($content: flex-start, $items: stretch, $direction: row, $wrap: wrap) {
  $packLegacy: $content;
  @if $packLegacy == flex-start {
    $packLegacy: start;
  } @else if $packLegacy == flex-end {
    $packLegacy: end;
  }

  $alignLegacy: $items;
  @if $alignLegacy ==flex-start {
    $alignLegacy: start;
  } @else if $alignLegacy == flex-end {
    $alignLegacy: end;
  }

  $oritentLegacy: $direction;
  $directionLegacy: normal;
  @if $oritentLegacy == row {
    $oritentLegacy: horizontal;
  } @else if $oritentLegacy == column {
    $oritentLegacy: vertical;
  }

/** SAFARI **/
  display: -webkit-box;
  -webkit-box-orient: $oritentLegacy;
  -webkit-box-pack: $packLegacy;
  -webkit-box-align: $alignLegacy;
/** FIREFOX LEGACY **/
  display: -moz-box;
  -moz-box-orient: $oritentLegacy;
  -moz-box-direction: $directionLegacy;
  -moz-box-pack: $packLegacy;
  -moz-box-align: $alignLegacy;
/** LEGACY **/
  display: box;
  box-orient: $oritentLegacy;
  box-direction: $directionLegacy;
  box-pack: $packLegacy;
  box-align: $alignLegacy;
/** IE 10+ **/
  display: -ms-flexbox;
  -ms-flex-wrap: $wrap;
  -ms-flex-direction: $direction;
  -ms-justify-content: $content;
  -ms-align-items: $items;
/** CHROME **/
  display: -webkit-flex;
  -webkit-flex-wrap: $wrap;
  -webkit-flex-direction: $direction;
  -webkit-justify-content: $content;
  -webkit-align-items: $items;
/** NATIVE **/
  display: flex;
  flex-wrap: $wrap;
  flex-direction: $direction;
  justify-content: $content;
  align-items: $items;
} //@mixin flex

@mixin flexItem($width) {
  -webkit-box-flex: $width;
  -moz-box-flex: $width;
  box-flex: $width;
  -ms-flex: $width;
  -webkit-flex: $width;
  flex: $width;

  min-height: 0;
}
```

Firefox however only half supports flexbox (all revisions) and to get around this I would recommend using [Modernizr](http://modernizr.com/ "Modernizr") as this will add the class "no-flexbox" to the <html> tag. This provides us with a simple work around that allows non flexbox supporting browsers render correctly by using specifically crafted and targeted CSS for non-flexbox browsers

I found that IE9 support could be implemented using the [flexie](http://flexiejs.com/ "FlexieJS") javascript plugin. In IE8 M[odernizr](http://modernizr.com/ "Modernizr") will add the class "no-flexboxlegacy" which can again allow you to create targeted CSS that wont affect your Flexbox layout.

For a great overview of the "CR" of flexbox, CSS Tricks has an amazingly comprehensive coverage of the functionality here [http://css-tricks.com/snippets/css/a-guide-to-flexbox/](http://css-tricks.com/snippets/css/a-guide-to-flexbox/)
