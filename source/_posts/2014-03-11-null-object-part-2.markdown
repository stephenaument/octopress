---
layout: post
title: "Null Object Part 2"
date: 2014-03-11 14:57:10 -0600
comments: true
categories:
--

In a [previous post](/blog/2014/03/05/the-null-object-pattern-and-method-missing-in-ruby/) I
described using `method_missing` in a Null Object to stand in for another
object and respond smartly to the original object's interface. At the end of
that post I suggested that you could go further than we did in creating a more
abstract object. Well, we got that chance.

In this post I will show you how we created a `NilObject` that can both stand
on its own and be extended with added functionality in a subclass. Our
`NilObject` needs to be smart about the interface of the class that it's
faking, so we'll have to take that into account. Finally, we decided to
implement a smart `respond_to?` method to round out the functionality of our
`NilObject`.

Here's the form that our NilDuck took at the end of the previous post:

{% gist 9422568 nil_duck_previous.rb %}

A few days after making those changes, we ran into another custom Null Object
in our app, the `NilGoose`. Our `NilGoose` ran into the
same problem that `NilDuck` originally did. A new method was added to the
`Goose` class that wasn't reflected in `NilGoose` and a bug appeared.

What to do? Well, we try not to run ahead of ourselves or our tests, so the
first thing we did was create a test that expected the missing method, watched
it fail, and copied the `method_missing` definition from `NilDuck`, renaming as
we needed to.

{% gist 9422568 nil_goose_initial.rb %}

This made our test happy. But the duplicated code made us sad. We like
[DRY](http://en.wikipedia.org/wiki/Don't_repeat_yourself) code around here. We
knew it was time to factor out our common code into a single spot.

We thought about putting it in a Module that could be included into other
special null objects, but decided instead to make it a full-fledged `NilObject`
class that we could use standalone as well as a superclass for our `NilDuck`
and `NilGoose`.

{% gist 9422568 nil_object_1.rb %}

If you recall in our `method_missing` implementation, we had to new up an
instance of our `Duck` class to determine whether it responds to the method we
are testing. Since we want to reuse this method in the generic `NilObject`, we
won't know which class to check. Our first pass at a solution requires the
class to be passed into the `NilObject` constructor, which would look like
`faker = NilObject.new(Duck)`.

In our `initialize` method we save the class passed in into an instance
variable, `@klass`, which `method_missing` then uses to new up an instance to
check. Now we can fake any class we want with our `NilObject`, and it will
handle exactly those methods the subject responds to and no others. (Note: If
you're weirded out by passing a class around like that, remember that
everything in Ruby is an object, including classes)

Now we can make `NilDuck` inherit from `NilObject`:

{% gist 9422568 nil_duck_final.rb %}

and the same for `NilGoose`:

{% gist 9422568 nil_goose_final.rb %}

This is all very cool, but if I leave the code like this I'm going to have to
run around and find every place in the app in which I create an instance of
`NilDuck` or `NilGoose` and add the new argument. Hopefully it's not very many
places, but this is not something I want to do. Let's go ahead and add
initialize methods to our `NilDuck` and `NilGoose` classes to take care of
that:

{% gist 9422568 nil_duck_intermediate.rb %}

and

{% gist 9422568 nil_goose_intermediate.rb %}

That works, and it keeps us from having to spread changes around the codebase,
but it does add to the duplication a bit. That initialize method needs to be
added to every new subclass in order to get the shorter `NilSomething.new`
syntax. We could live with this, or we could find a way to bake that smarts
into `NilObject` itself. But how?

My pair for the week, [@timtyrell](https://twitter.com/timtyrrell) came up with
the idea of keying off of the classname of the `NilObject` subclass. And that's
exactly what we did, we made the class parameter optional and keyed off of the
classname.

{% gist 9422568 nil_object_2.rb %}

The `real_class_name` method (naming is hard; this should probably be
`subject_class_name` or something) takes the name of the class, `NilDuck` or
`NilGoose` and takes the substring that skips the first 3 letters (the length
of 'Nil') to the end. It then turns it into a constant so it can be
instantiated.

As an added bonus we decided to implement `respond_to?` so it behaves exactly
like the subject class would behave.

{% gist 9422568 nil_object_3.rb %}

So there you have it. We now have a `NilObject` that can stand in for any other
object like so: `NilObject.new(MyOtherClass)` and it will respond to exactly
the same interface as `MyOtherClass` and answer `respond_to?` just like an
instance of `MyOtherClass` would. If we need to override some of the methods of
`MyOtherClass` to return other than `nil` or `false` we can create a new
`NilMyOtherClass` as a subclass of `NilObject` and override what we need to.

I'll leave you with a thought exercise. What would happen if you instantiated a
`NilObject` without passing a class to the constructor: `nil_object =
NilObject.new`? What would you get? As a bonus, how would you make the code
inside the `real_class_name` method more intention revealing than what we wrote?
Leave a comment if you know the answers.
