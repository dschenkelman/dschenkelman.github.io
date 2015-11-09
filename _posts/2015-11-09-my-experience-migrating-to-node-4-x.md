---
layout: post
title: 'My experience migrating to node 4.x'
categories:
- io.js
- hapi.js
- node4
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
**tl;dr**: Migrating to node.js 4.x is not difficult, but can be a bit tedious as it involves updating a lot of dependencies in medium and large size projects.

-------------

At Auth0, our API v2 is a medium size node.js project that uses hapi.js. When we first deployed it, it was running node.js 0.10.x.

Since this wasn't a large project and it was relatively new, we decided to use it to experiment with some new things, including io.js (at that point it was in version 1.0.4). io.js was using a more modern version of v8 than 0.10.x, which meant that by making the change we automatically got a big increase in the amount of requests per second that we could handle.

However, migrating to io.js took a bit more time than what we had expected at the beginning. After the migration, some of our automated tests were failing, so we couldn't stick with the change. It took us some time to come up with a definitive set of repro steps, but eventually it turned out to be a [bug in hapi.js](https://github.com/hapijs/hapi/issues/2427#issuecomment-75768242) that was present in some versions of node 0.10.x, and all versions of 0.12.x and iojs. The bug was quickly fixed in hapi.js and on March 11th 2015, we were able to deploy our API running on io.js for the first time :)

Some months later, on July 5th 2015 and July 13th 2015, we had to update this component to io.js versions [1.8.3](https://nodejs.org/en/blog/weekly-updates/weekly-update.2015-07-03/) and [1.8.4](https://nodejs.org/en/blog/weekly-updates/weekly-update.2015-07-10/) respectively due to some security vulnerabilities.

More recently, we upgraded some of our other larger components to io.js 1.8.4. This mostly involved getting the right version of `nan` to be used to build the native dependencies. While doing this we found a bit of an inconvenience: there was an [issue](https://github.com/nodejs/node-v0.x-archive/issues/9261) with using UDP sockets together with the `cluster` module. We eventually stopped using statsd, so we no longer needed UDP sockets and were able to complete the migration.

Some weeks ago, I had a bit of idle time during a weekend, and started a proof of concept to see what it would take to get our API v2 migrated from 1.8.4 to 4.x.x. It mostly involved changing a lot of nested dependencies for native modules, and after updating all of them I was able to get everything working.

One of the things I found is that version 1.4.x of the `mongodb` module can't use the native BSON extensions when using node 4.x.x. For that reason, we decided to start using version 2.x.x of the `mongodb` module before migrating to node 4.x.x, which required some changes to our code to deal with some of the module's updates.

After migrating to version 2.x.x of the `mongodb` module, we found a [bug with reconnections](https://github.com/christkv/mongodb-core/pull/54) that resulted in uncaught exceptions. I've sent a PR that has been merged and I'm waiting for a new version with the fix to be released.

# Conclusion
All in all, the path to node.js 4.x.x is not an particularly tough one, but it has a lot of detours.

One key takeway from this, is that having LOTS automated when doing this kind of updates is really important, as it can help you catch errors early in the update process.