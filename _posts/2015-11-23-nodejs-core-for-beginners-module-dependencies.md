---
layout: post
title: 'node.js core for beginners: module dependencies'
categories:
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

Last Thursday I had the chance to see [Thorsten Lorenz](https://twitter.com/thlorenz) present at a meetup about [debugging and profiling node.js](http://thlorenz.com/talks/debugging-profiling-nodejs/book/). At one point during the talk he mentioned that he started to learn about how node.js works by reading the [source code](https://github.com/nodejs/node) of one module every day during lunch, beginning with the JS code (**lib** folder) and the moving on to the C++ code (**src** folder).

I'm probably not going to have that much discipline, but on Saturday when I read the code for [crypto](https://github.com/nodejs/node/blob/master/lib/crypto.js) I found that it might have been useful to read the code for its dependencies before, in order to better understand how it worked.

Being who I am, I started to look for ways to create dependency graphs for the  node.js source code, so I could come up with a better order to read the source code files. I quickly found a great node module named [MaDGe](https://github.com/pahen/madge), which parses the AST for a set of modules and automatically creates a graph with the dependencies.

So, I started to play with it a bit...

## Playground
I thought I would take a look at the modules with the most dependencies:
{% highlight bash %}
➜  node git:(master) madge --summary lib | head -n 5
crypto: 9
http: 9
repl: 9
net: 9
_tls_wrap: 9
{% endhighlight %}

and the modules with the least dependencies:
{% highlight bash %}
➜  node git:(master) madge --summary lib | tail -n 5
internal/util: 0
internal/readline: 0
v8: 0
vm: 0
constants: 0
{% endhighlight %}

Turns out starting out with `crypto` might not have been the best idea in terms of dependencies :). I could also easily find the all dependencies for any module:
{% highlight bash %}
➜  node git:(master) madge lib/crypto.js
crypto
  assert
  buffer
  constants
  internal/streams/lazy_transform
  internal/util
  stream
  string_decoder
  tls
  util
{% endhighlight %}

And there is also a great option that allows you to create an image with the module's dependency graph (open it in a new tab, it's really cool):
<figure>
  <img src="{{ site.url }}/images/posts/20151123/node.png" alt="node modules dependency graph">
</figure>

## Including native modules
MaDGe uses [detective](https://github.com/substack/node-detective), which supports an `isRequire` callback to determine if an AST node is a `require` call. By making a simple (hacky) change to the code, you can also get it to include native dependencies. All it takes is changing the `detective(fileData.src)` invocation [here](https://github.com/pahen/madge/blob/master/lib/parse/cjs.js#L56) to:
{% highlight javascript %}
detective(fileData.src, {
    isRequire: function(node){
        if (node.callee.type === 'MemberExpression' &&
            node.callee.object.name === 'process' &&
            node.callee.property.name === 'binding'){
            // this is a quick hack :)
            if (node.arguments[0]){
                node.arguments[0].value += ' (native)';
            }

            return true;
        }
        return (node.callee.type === 'Identifier' &&
            node.callee.name === 'require');
    }
})
{% endhighlight %}

And this is the resulting dependency graph after the update:
<figure>
  <img src="{{ site.url }}/images/posts/20151123/native.png" alt="node modules dependency graph (including native)">
</figure>

## Conclusion
If you are set on reading node's source code and, like me, prefer to have a certain order when diving deep into things, you can check play around with MaDGe to figure out how to get started.