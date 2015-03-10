---
layout: post
title: 'patova: simple throttling for hapi.js'
categories:
- hapi.js
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
**tl;dr**: If you don't want to read the post check out the sample [here](https://github.com/dschenkelman/patova/tree/master/samples/basic_auth).

-------------

If you are building an API, it is likely that at some point you will have the need to throttle requests. There might be several reasons for that but the core part is that you want to limit the amount of requests that a certain party (could be a user, ip, etc.) can perform over a period of time.

At Auth0 we are working on [limitd](https://github.com/auth0/limitd), which implements the [token bucket](http://en.wikipedia.org/wiki/Token_bucket) algorithm to support throttling scenarios. Since our [API v2](https://auth0.com/docs/apiv2) uses [hapi.js](http://hapijs.com), we also decided to create [patova](https://github.com/dschenkelman/patova), a limitd plug-in for hapi.js.

This post shows how easy it is to enable this feature in your application.

## Requirements
Let's assume that we have an API that uses basic authentication and we want to limit requests by user.

## Implementation
Create a new directory and install the required modules:
{% highlight bash %}
> npm i -S hapi patova hapi-auth-basic
{% endhighlight %}

Then create a file named `server.js` with the following code (the sample is based on the [hapi-basic-auth](https://github.com/hapijs/hapi-auth-basic) readme):
{% highlight javascript %}
var Hapi = require('hapi');

var users = {
  john: {
    username: 'john',
    password: 'secret',
    name: 'John Doe',
    id: '2133d32a'
  }
};

var server = new Hapi.Server();
server.connection({
  host: 'localhost',
  port: 8000
});

var validate = function (username, password, callback) {
  var user = users[username];
  if (!user) { return callback(null, false); }

  var isValid = password === user.password;
  callback(null, isValid, { id: user.id, name: user.name });
};

var plugins = [ require('hapi-auth-basic') ];

server.register(plugins, function (err) {
  server.auth.strategy('simple', 'basic', { validateFunc: validate });
  server.route({
    method: 'GET',
    path: '/',
    config: {
      auth: 'simple',
      handler: function(request, reply){
        reply(request.auth.credentials.name);
      }
    }
  });
});

server.start();
{% endhighlight %}

Start the server `node server.js` and perform a request to make sure everything is working:
{% highlight bash %}
> curl http://127.0.0.1:8000 -H "Authorization: Basic am9objpzZWNyZXQ="
John Doe
{% endhighlight %}

Create a file named `limitd.config` to hold limitd's configuration:
{% highlight yaml %}
#port to listen
port: 9001

#path where the data will be stored
db: /tmp/hapi-throttle-example.db

#Define the types of buckets
buckets:
  user:
    size: 5
    per_second: 5
{% endhighlight %}

And start the limitd server:
{% highlight javascript %}
> node_modules/patova/node_modules/limitd/bin/limitd --config-file limitd.config
{% endhighlight %}

Finally, add the patova plug-in to the hapi server:
{% highlight javascript %}
var patovaConfig = {
  register: require('patova'),
  options: {
    event: 'onPostAuth',
    type: 'user',
    address: { host: '127.0.0.1', port: 9001 },
    extractKey: function(request, reply, done){
      var key = request.auth.credentials.id;
      done(null, key);
    }
  },
};

plugins.push(patovaConfig);
{% endhighlight %}

That's all there is to it! 

## Trying it out
Let's make sure everything is working. Restart the server and perform 6 requests (remember we configured 5 req/sec as the limit):
{% highlight bash %}
for i in {1..6}                                                      
do
  curl http://127.0.0.1:8000 -H "Authorization: Basic am9objpzZWNyZXQ=" | xargs echo
done
{% endhighlight %}

The last one should display the following
{% highlight bash %}
{statusCode:429,error:Too Many Requests}
{% endhighlight %}

As you can see, adding throttling to you hapi.js API is very simple. If you have questions or comments feel free to post them here or as issues in the project.