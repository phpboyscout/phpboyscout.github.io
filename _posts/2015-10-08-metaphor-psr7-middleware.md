---
title: "A metaphor about PSR-7 and Middleware for non-developers"
date: "2015-10-08"
---

Never one to shy away from coming up with a metaphor for explaining something technical I found myself having to come up with one on the spot for PSR-7 and Middleware while at the recent PHPNW15 Conference.

Normally my brain will come up with something completely inappropriate but this time round I found I quite liked the imagery that came to mind.

If you would like to find out more of the specifics about PSR-7 you can take a look at [http://www.php-fig.org/psr/psr-7/](http://www.php-fig.org/psr/psr-7/) which will make a better job of explaining it than I could ever do.

Now on to the metaphor <!--more-->

Imagine a house on fire, a bizarre way to start I know but bear with me. The nearest well with water that can put out the fire is 500 meters away! We then have a human chain stretching between the well and the house with a bucket going back and forth between trying to put the fire out. ![fire-bucket-brigade](/assets/images/fire-bucket-brigade.jpg) So lets break this down, the house represents the internet, or more specifically you and your browser. The fact you are on fire means that you are desperately needing water to quench the flames. At this point you send an empty bucket which represents your "request", along the human chain, which in itself represents the application, to the well.

At the start of the chain the bucket is pretty normal, it's a bucket of course, its round, made of wood with a rope handle, lets say it has a small leak in it.

As it travels down the chain it's passed from person to person, everyone in it has the opportunity to do something with the bucket, or not as the case may be and could just pass it to the next person in the chain. Others may attempt to fix the leak in the bucket, someone may choose to replace it with a metal bucket, change the handle or make it bigger. Regardless of what may be done to the bucket in essence it remains a bucket.

![colonial_bucket3](/assets/images/colonial_bucket3.jpg)

Inexorably the bucket will continue to move down the chain to the well. When it reaches the well it changes state because now it has been filled with water. All of the interaction with the bucket thus far, mean that what happens at the well could vary depending on the changes have been made . If its been made bigger, for example, it could be filled with significantly more water, if swapped for a metal one it could imply that the bucket descends the well to get the water quicker because its heavier. Either way it is filled with water and begins its journey back towards the house.

Again it passes through the hands of each person in the chain, but now that its state has changed it now has the opportunity to be modified again. Someone may empty some water out as there is too much in the bucket, others may say that there is not enough and send it back down the line towards the well to be refilled. Either way the bucket continues to change hands over and over until it reaches the house and the contents are thrown on the fire to complete the request for water.

During this whole time the human chain could have been in flux. Some people may have swapped places, left the chain, added to the chain, some extraordinary people may have played leapfrog in the chain and appeared to handle the bucket more than once. Regardless of these changes the chain remains and continues to pass the bucket from one person to the another as long as the requests for water keep coming.

This, in the simplest possible form, explains PSR-7 and the concept of Middleware.

The bucket remains a bucket because PSR-7 says that is what is needed to complete the request for water, it also defines how you should interact with it regardless of what modifications have been made. If the bucket cant be used according to how PSR-7 describes a bucket to be, then the middleware can't complete the request.

Every person in the human chain can be classed as a piece of middleware all the way from the house to the well and back again. If at any point someone enters the chain that doesn't agree that the bucket is a bucket or doesn't know how to handle it, then the it is dropped on the ground and the request fails.
