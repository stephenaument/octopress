---
layout: post
title: "Backbone Collection Testing Gotcha"
date: 2014-03-12 16:46:04 -0500
comments: true
categories:
  - Backbone
  - Javascript
  - Coffeescript
  - Debugging
---

Be careful of this gotcha when testing a [Backbone.js](http://backbonejs.org/)
collection. I don't have much experience with Backbone, so this is probably
obvious to more experienced users, but hopefully this will help somebody else,
or at least myself in the future.

Today I needed to test the behavior of a Collection with a certain number of
Models in it. I didn't care about the content of the models, just that they
contained a certain field. So, my pair and I came up with this setup to run
in our [Jasmine](http://jasmine.github.io/) specs:

{% gist 9541718 initial_spec.js.coffee %}

We created an array of dummy models and passed them into our Collection. This
is one way you can create and populate a Collection. You pass in your array of
models and they become the models in the Collection.

> If that `providerArray = (provider for [1..18])` syntax freaks you out, don't
> worry. It's just a bit of fancy CoffeeScript and all you need to understand
> for now is that it creates an 18 item array with each element being
> `provider`.

Makes sense, right? Well, our test failed and when we started debugging in the
browser, here's what we saw:

{% gist 9541718 initial_result.js %}

What??? `length: 1, models: Array[1]`? When we looked into the models array it
contained a single item from the array we passed in.

Next we tried setting the models array after the fact:

{% gist 9541718 intermediate_spec.js.coffee %}

The models array seems right now, but the collection length was still one:

{% gist 9541718 intermediate_result.js %}

This was just as baffling. By this time we had called in [Tim
Tyrell](@timtyrrell) to tell us what we were doing wrong. Tim had the foresight
to read, not just the docs for the Backbone Collection `initializer` method, but
also the documentation for the `add` method, which had this to say (emphasis
added):

> *If you're adding models to the collection that are already in the collection,
> they'll be ignored*, unless you pass {merge: true}, in which case their
> attributes will be merged into the corresponding models, firing any
> appropriate "change" events.

Interesting! When add is called, it ignores any objects that are already
present in the models array. Since we are using the same model instance in each
position of our array, only the first one is being stored. The rest are being
dumped.

A quick check of the [Backbone source
code](https://github.com/jashkenas/backbone/blob/master/backbone.js#L785-L799)
confirms that the initialize method is calling `reset` with the array you pass
in, which iterates over the array and calls `add` for each.

{% gist 9541718 backbone_reset.js %}

So, my pair, Curtis Ekstrom decided to try creating a new object for each
element of the array with the same data to see if that would work.

{% gist 9541718 final_spec.js.coffee %}

And so it did. At first I thought that Backbone was performing a simple object
identity check (===) vs an equality check (==). That would make sense of the
behavior we saw, but when I read the `add` documentation again, I saw that
passing `{merge: true}` along with the model or model array would result in the
attributes passed in being merged into the existing objects. It must be doing
something else entirely.

Another peek into the [source
code](https://github.com/jashkenas/backbone/blob/master/backbone.js#L718-L728)
confirms. `add` calls `set`, which contains the relevant:

{% gist 9541718 backbone_set.js %}

That first line `if (existing = this.get(id)) {` is the key. It looks for an
existing model in the `_byId` object hash, with the same `id`, `cid` or that is the
object itself. If it finds it, and `merge` is not `true` it rejects the model.

TL;DR - Make sure each Model in the array you pass to the Collection
initializer is a distinct instance, even if the data is otherwise identincal.
