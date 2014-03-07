---
layout: post
title: "Null Object Part 2"
date: 2014-03-11 14:57:10 -0600
comments: true
categories: 
--

In a [previous post](/blog/2014/03/05/the-null-object-pattern-and-method-missing-in-ruby/) I described using `method_missing` in a Null Object to stand in for another object and respond smartly to the original object's interface. At the end of that post I suggested that you could go further than we did in creating a more abstract object. Well, we got that chance.

In this post I will show you how we created a `NilObject` that can both stand on its own and be extended with added functionality in a subclass. Our `NilObject` needs to be smart about the interface of the class that it's faking, so we'll have to take that into account. Finally, we decided to implement a smart `respond_to?` method to round out the functionality of our `NilObject`.

Here's the final form that our NilDuck took at the end of the previous post:

###GIST HERE

A few days after making those changes, we ran into another custom Null Object in our app, the `NilGoose` (shhh...let it happen). Our `NilGoose` ran into the same problem that `NilDuck` originally did. A new method was added to the `Goose` class that wasn't reflected in `NilGoose` and a bug appeared.

What to do? Well, we try not to run ahead of ourselves or our tests, so the first thing we did was create a test that expected the missing method, watched it fail, and copied the `method_missing` definition from `NilDuck`, renaming as we needed to. This made our test happy. But the duplicated code made us sad. We knew it was time to factor out our common code into a single spot.

We thought about putting it in a Module, but decided instead to make it a full-fledged `NilObject` class that we could reuse anywhere else we might need.

###GIST HERE


