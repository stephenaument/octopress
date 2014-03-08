---
layout: post
title: "The Null Object Pattern and method_missing in Ruby"
date: 2014-03-05 23:01:09 -0600
comments: true
categories:
---

I encountered an interesting problem this week that allowed me to dig into the Null Object pattern, Ruby duck typing, and Ruby's method_missing.

In our app we have a `Duck` class (name changed to protect the innocent). Most of our users have a `Duck` associated with their account, but some don't. In our application, we have certain situations which expect a Duck, even if the user doesn't have one. If we don't have a Duck, but our app expects one and passes it a message like `quack,` we will receive an ugly `NoMethodError: undefined method 'quack' for nil:NilClass` error.

How can we solve this without spreading smelly nil checks around our system? It turns out there is an existing design pattern for just this situation, the Null Object pattern. In the Null Object pattern, we create a stand in object for our `Duck` that can respond to the same interface, but without returning any values. We avoid ugly `NoMethodError`s without having to check for the objects existence everywhere.

Here is our Duck:

{% gist 9426713 duck1.rb %}

This is a Ruby on Rails ActiveRecord model, so there is more to it than we see here. Here is the database migration file:

{% gist 9426713 create_ducks_migration1.rb %}

For each of these fields, ActiveRecord will create getter and setter methods in the background. But for the `migratory` boolean field, we will get the bonus `migratory?` predicate method.

In our case, it turned out that we need our `NilDuck` to return *some* values, but most things should just return `nil`. The simplest way for us to implement our `NilDuck`, and the state in which I found my `NilDuck` code this week looks something like the following:

{% gist 9426713 nil_duck1.rb %}

Here we have provided the attribute setters our fake object needs with the `attr_writer` call. We then proceed to override each getter as needed. There are a few special cases in which a value other than nil is desired, but the default return value for our `NilDuck` is `nil`. In the cases in which we don't have an actual `Duck` and need to use a demo `Duck`, we can pass in an instance of our `NilDuck` and everything will work as expected.

>Note: since Ruby is not a strongly typed language, we don't have to subclass our `Duck` here or implement a formal common interface as we would in a strongly-typed language like Java. In Rubyland, types are determined by the public interface of the object in question. So if our `NilDuck`, or any other object for that matter, implements the methods our client object calls in `Duck`, we can substitute it. This is often called "duck typing," since if an object quacks like a duck, and acts like a duck, we can usually consider it a duck. We *do* implement a common interface, but the interface that we implement is conceptual, represented by the messages to which we expect our duck to respond. It's not baked into the language as a formal construct.

So what's wrong with this code? Nothing at face value. Maybe it could be refactored into something less verbose, but it's functional and fairly straightforward. In the beginning it served our app quite well. But I came to this code last week because of a bug uncovered by a change in another area of the app. It turned out that my `NilDuck` code had gotten stale and not kept up with changes to the `Duck` class. Methods had been added to `Duck`, but not to `NilDuck`. Now `Duck` looks like this:

{% gist 9426713 duck2.rb %}

And the migration:

{% gist 9426713 create_ducks_migration2.rb %}

We don't have equivalent `secondary_color` and  `multi_colored?` methods and our NilDuck started causing errors. Since none of these new methods requirs special values, I could have just added equivalent getters in NilDuck that returned `nil`. But since I'm a lazy programmer, I asked myself how I could avoid editing this class in the future. I turned to Ruby's `method_missing`!

In order to understand what my pair and I did here, you need to understand a little about the way Ruby looks up a method, or message, passed to an object. When you call a method on a Ruby object, Ruby first looks to see if the method has been defined on the object instance itself (or more precisely on it's [eigenclass](http://en.wikipedia.org/wiki/Metaclass)), if it doesn't find it it will then look in any included modules in reverse include order, then in the instance methods of the class, and finally in the instance methods of the `Object` class, the ultimate ancestor of every object in Ruby. If Ruby still can't find the method, it starts the search over at our instance, this time trying to call `method_missing`. If it can't find a definition for `method_missing` somewhere in our inheritance chain, only then will it throw a `NoMethodError`.

The `method_missing` callback gives us the hook we need to handle our, well, missing methods. Let's refactor our `NilDuck` to use `method_missing`:

{% gist 9426713 nil_duck2.rb %}

See what we did there? We want `NilDuck` to have the same interface as `Duck`, so in our `method_missing` implementation, we first checked to see whether `Duck` responds to the method our client code attempted to call. If it does, we return `nil`, otherwise we pass the buck to super. Our `NilDuck` will now respond to everything that `Duck` responds to and throw a `NoMethodError` for anything that it doesn't respond to. As a bonus, we got to eliminate a lot of code here.

But what about predicate methods? Well, in Ruby, `nil` is falsey, so this code will work correctly in an `if/unless/!` situation, but what about a situation in which you might need `migratory?` to explicitly give you `false` instead of `nil`? What if you are going to pass that `true` or `false` along to someone else as a string? What if you need to feed it into javascript from an erb or haml template?

{% gist 9426713 javascript.js.rb %}

If this `duck` is our `NilDuck`, `duck.multi_colored?` will return `nil`. When the template is rendered and the string is interpolated, `nil`'s `to_s` method will be invoked, which will return an empty string instead of `'false'`. This will result in a javascript error in the browser:

{% gist 9426713 javascript1.js %}

Whoops! That's not what we wanted at all! We want `'true'` or `'false'` to be rendered. How can we handle that? Well, we can fall back and explicitly create a `multi_colored?` method in our `NilDuck` that returns `false`, but we can do better than that. Let's stay lazy so we don't have to fix this again! Since it turns out that we don't really want our predicate methods to return just a falsey value, but `false` itself. Let's account for that:

{% gist 9426713 nil_duck3.rb %}

Now if `Duck` responds to our method and our method ends with '?', `NilDuck` will return `false`, if it doesn't end with '?' it will return `nil`. Our template will now render the javascript we expected:

{% gist 9426713 javascript2.js %}

You could certainly take this further and abstract out a generic `NilObject` for your app, but this serves our needs quite well. For further reading on the Null Object pattern: [http://devblog.avdi.org/2011/05/30/null-objects-and-falsiness/]
