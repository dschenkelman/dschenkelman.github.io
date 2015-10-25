---
layout: post
title: 'Feature changes at Auth0'
categories:
- auth0
- node
- feature-changes
status: publish
comments: true
type: post
published: true
author:
  login: dschenkelman
  email: damian.schenkelman@gmail.com
  display_name: Damian Schenkelman
  first_name: Damian
  last_name: Schenkelman
---
**tl;dr**: We created a [node module](https://github.com/dschenkelman/feature-change) to run two versions of a feature at once (original and new), compare the results and always return the original results. It is useful to deploy new versions of features without breaking changes.

-------------

At Auth0 we move fast, and I mean really fast. Yesterday alone we performed  14 deployments to production.

This is something I really like and a positive thing in general, as for example it allows us to [fix production bug in short timeframes](https://twitter.com/trydis/status/642809967859380224). Unfortunately, as we are all human, having so many deployments per day has also been the cause of unintentional breaking changes.

To mitigate this, one option would be to move slower. Another option would be to start relying on more tools and automated tests to make sure that deployments do not introduce any breaking changes.

We decided to do both. On the one hand, by providing a cluster with only one deployment per day (with changes from the previous day) for some of our paid customers who request it. On the other hand, by improving our automated tests and performing additional checks before new versions are deployed to production.

One of the tools we created is a node module named [feature-change](https://github.com/dschenkelman/feature-change). We got the idea from a blog post from [@holman](https://twitter.com/holman), which explains how GitHub ["moves fast & breaks nothing"](http://zachholman.com/talk/move-fast-break-nothing/). I think it is better to understand how it works with a real life example.

# Moving search from mongo to ES
In the beginning, there was a request from a single customer:

> Is there a way to search for users by field XXX?

There wasn't, so we went ahead and implemented search against our small MongoDB instance. And all was good.

After that came another customer:

> Is there a way to search for users by field YYY?

And then another one. So we decided to allow searching by any field, providing some sensible defaults for the common cases. For a time this worked perfectly, but as our customer base (fortunately) grew, we realized MongoDB wasn't exactly build for this kind of thing.

For v2 of our API we had learnt our lesson, so we did not allow searching against MongoDB and instead used Elastic Search for this use case, which so far is working as expected.

The thing is that some of our customers are still on v1 of our API. Some of their searches against MongoDB were really slow and affected the database's stability, which is why we decided to exactly replicate the v1 search functionality in ES to reduce the load on our database.

# Introducing feature-change
We know a LOT of our customers rely on search, so we didn't want to introduce any issues with this change. For that purpose we decided to take advantage of the idempotency property of a search operation to deploy a temporary version to production that did all of the following:

* Perform the search against MongoDB
* Perform the search against ES
* Compare the results and write them to our logs if there was a difference
* Always return the MongoDB result

Using `feature-change` it looked like this:
{% highlight javascript %}
var feature_change = require('feature-change');

var current_implementation = mongo_search;
var new_implementation = es_search;

feature_change(function(cb){
    current_implementation(current_opts, cb);
}, function(cb){
    new_implementation(es_opts, cb);
}, function(mongo_result, es_result){
    // invoked when there is a difference in the results (useful for logging)
}, function(err, result){
    // this is the original callback we were using for mongo
    // err and result always come from mongo_search
});
{% endhighlight %}

Once we deployed this version to production we started to monitor our logs to find errors. In the following chart, the blue part of the bar represents the difference percentage between both results.
<figure>
  <img src="{{ site.url }}/images/posts/20151026/differences.png" alt="Differences">
</figure>

After fixing three different bugs we got rid of all errors causes. We monitored the differences for a week to see if any new sources of errors appeared. When they didn't, we removed `feature-change` and the Mongo search from the version in production.

# Show me the code
We have [open sourced](https://github.com/dschenkelman/feature-change) `feature-change` so you can make use of it in your own applications. Feel free to open issues, send PRs, etc.