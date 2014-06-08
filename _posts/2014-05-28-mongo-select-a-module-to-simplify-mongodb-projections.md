---
layout: post
title: 'mongo-select: A module to simplify mongoDB projections'
categories:
- mongodb
- node.js
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
Today I started to work on adding field selection capabilities to some of the endpoints used by the [Auth0 app](http://app.auth0.com). Part of that involves updating the mongodb queries to [limit the fields returned by them](http://docs.mongodb.org/manual/tutorial/project-fields-from-query-results/).

As this work will probably span a set of queries, I decided to create a module to simplify the creation of the projection objects used for the queries. That's how **mongo-select** came to be:

* [Github site](https://github.com/dschenkelman/mongo-select)
* [NPM site](https://www.npmjs.org/package/mongo-select)

As the mongodb [documentation](http://docs.mongodb.org/manual/tutorial/project-fields-from-query-results/) clearly explains, you can either

* Determine a set of fields to include by specifying their name and a related _truthy_ value `{ name: true }`.
* Determine a set of fields to exclude by specifying their name and a related _falsy_ value `{ name: false }`.

**mongo-select** allows you to easily specify fields to include or exclude, both on a permanent and one time basis.

Most of the rest of this post comes from the [README.md](https://github.com/dschenkelman/mongo-select/blob/master/README.md) at the time of writing.

## Installing
To install the latest version simply run:
{% highlight bash %}
npm install mongo-select
{% endhighlight %}

## Basic example
The following sample code shows how you can work with the [native mongodb NodeJs driver](https://github.com/mongodb/node-mongodb-native) to exclude the _user_ field:
{% highlight javascript %}
var select = require('mongo-select').select();
var mongodb = require('mongodb');

var MongoClient = mongodb.MongoClient;

MongoClient.connect('mongodb://127.0.0.1:27017/test', function(err, db) {
  if(err) throw err;

  var users = db.collection('users');
  scripts = users.find({}, select.include(['name', 'email']), 
    function(err, result){
      // code here, access to only result[i]._id, result[i].name and result[i].email
    });
});
{% endhighlight %}

## Including fields
To include fields use the following:
{% highlight javascript %}
var select = require('mongo-select').select();

var projection = select.include(['name', 'email', 'children.name']);

console.log(projection); // { 'name': false, 'email': false, 'children.name': false };
{% endhighlight %}

## Excluding fields, no _id and chaining
You can also exclude fields using the exclude method and the `_id` field using `noId()`. You can chain exclusions/inclusions and `noId()`. One thing to consider is that in order to provide a fluent interface the chaining methods begin with `_`. Otherwise this might affect documents with fields named _exclude_, _include_, _noId_.
{% highlight javascript %}
var select = require('mongo-select').select();

var projection = select.exclude(['name'])._exclude(['email', 'children.name'])._noId();

console.log(projection); // { '_id': false, 'name': false, 'email': false, 'children.name': false };
{% endhighlight %}

## Permanent exclusion/inclusion
Sometimes it is important to always exclude or include a set of fields. That means that if they were permanently excluded and then specifically included they won't make it into the projection and viceversa:
{% highlight javascript %}
var select = require('mongo-select').select();

select.exclude(['name', 'email', 'children.name'])._always();

var exclusion = select.exclude(['address']);

console.log(exclusion); // { 'name': false, 'email': false, 'children.name': false };

var inclusion = select.include(['name', 'email']);

console.log(inclusion); // { };
{% endhighlight %}

To clear permanent registrations simply:
{% highlight javascript %}
select.clear();
{% endhighlight %}

## Wrapping up
I'm totally up for contributions via PRs or issues, so if you have some feedback about the package or improvement ideas feel free to jump in.
