---
layout: post
title: "Ruby Hash#fetch and Javascript"
date: 2014-03-06 19:21:15 -0600
comments: true
categories: 
---

In a [previous
post](/blog/2014/03/05/the-null-object-pattern-and-method-missing-in-ruby/) I
mentioned a case in which I had a Haml template in a Rails app with Javascript
in it. The template was rendering Ruby values into what would become a JSON
object (Javascript Hash) in the browser. In that particular case, a method call
on a Null Object was returning nil and breaking the Javascript.

Well, today I encountered another variation of this problem. In this case, I
had a Ruby Hash feeding values into the JSON object. It looked like this:

{% gist 9403295 %}

Two of these fields on the ruby hash are empty: `language` and `properties`.
Notice that this isn't a problem for most of these fields, but for one it is:

{% gist 9404094 %}

See that? The first four fields feed into double quotes as strings. If they are
empty, they are empty. It may cause an issue somewhere down the line, but the
Javascript itself is valid.

The `properties` field, on the other hand is expecting a JSON object. That
missing bit will cause Javascript to blow up once this loads in the browser.

What's the solution here? In this case, I turned to `Hash#fetch` rather than
the `[]` accessor. What's the difference? There are two main differences
between the bracket accessor and the `fetch` method. First,
`my_hash[:some_missing_key]` will return `nil` if the key is not found in the
hash. If `my_hash.fetch(:some_missing_key)` can't find a key, it will raise a
`KeyError` exception.

That's not really what we are looking for, which brings me to the second
difference. `Hash#fetch` takes an optional second argument which specifies a
default value. So if we call `my_hash.fetch(:some_missing_key, 'tacos')` and
the key is not found in the hash, it will return `'tacos'` instead of `nil`.
That's exactly what I needed. I returned an empty JSON object `{}` as the
default.

{% gist 9404014 %}

This degraded nicely in the cases in which I had no `properties` hash in my
`search_params` hash.

{% gist 9404101 %}

No more broken Javascript.
