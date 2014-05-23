---
layout: post
title: 'Creating friendly frozen object APIs using ECMAScript 6 Proxies'
categories:
- ECMAScript
- JavaScript
- Immutability
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
There is a lot of talk nowadays related to the benefits of immutability in programming languages. Functional languages such as [F# and Scala](http://www.tiobe.com/index.php/content/paperinfo/tpci/index.html) are gaining more popularity because of this. Commonly considered object oriented languages such as C# and Java go _more functional_ with each release, [Java including lambdas](http://docs.oracle.com/javase/tutorial/java/javaOO/lambdaexpressions.html) in version 8 (although that is not directly related to immutability) and C#  having the [immutable collections](http://msdn.microsoft.com/en-us/library/dn385366(v=vs.110).aspx) and new features in version 6 such as get only auto-properties, initializers for automatically implemented properties and primary constructors, which [can be combined together to write immutable classes/structures with less code](https://msmvps.com/blogs/jon_skeet/archive/2014/04/04/c-6-first-reactions.aspx).

Usually in those cases, the most highlighted benefit of immutability is that since you don't have to worry about mutable state you can write **lock free multithreaded code**, a thing that is becoming more important as the processing power of one CPU core is not going to continue growing in the future (unless until Quantum computers arrive, but I have no idea of how that would work). That benefit comes from the guarantees that computers can make about immutable state.

# Immutability in ECMAScript 5
ECMAScript 5 introduced the ability to make objects immutable through the use of `Object.freeze(obj)`. As the [documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze) explains, the method:

* Prevents the addition of new properties to the object.
* Prevents the removal of existing properties
* Prevents changes on any of the characteristics of existing properties.

_"In essence the object is made effectively immutable."_

Considering the current state of JavaScript writing multithreaded is not one of the goals for which this feature was designed. Nevertheless, considering JavaScript's dynamic nature I think it was done because:

* It should be potentially possible for engines to optimize code based on the knowledge that an object is immutable (although it seems [that is not the case right now](https://code.google.com/p/chromium/issues/detail?id=115960)).
* Sometimes it's easier to reason about things if you are sure they won't change.

# APIs for working with immutable elements
Considering all of the above, a fairly common scenario would be to want to create a new object similar to an existing one but changing some values from the original object. F# provides the following syntax for records through the usage of `with`:
{% highlight fsharp %}
type Point = {
  X: int;
  Y: int;
}

let point = { X = 0; Y = 0; }
let point2 = { point with X = 100; }
{% endhighlight %}

A similar API in C# would look like this (and some pains about it are documented in [this post](http://blogs.msdn.com/b/mostlytrue/archive/2014/05/12/misadventures-in-immutability.aspx)):
{% highlight csharp %}
public class Point {
    private readonly int x;
    private readonly int y;

    public Point(int x, int y){
        this.x = x;
        this.y = y;
    }

    public Point WithX(int x){
        return new Point(x, this.y);
    }

    public Point WithY(int Y){
        return new Point(this.x, y);
    }

    public int X { get { return this.x; } }
    public int Y { get { return this.y; } }
}

var p = new Point(0, 0);
var p2 = p.WithX(20);
{% endhighlight %}

To achieve the same thing in JavaScript we could create methods for each _field_, similar to the C# example. That could be either manual or automatic, the latter by calling a function passing the object as a parameter and creating a function for each _field_.

And then there's another option...

# Using Proxy to implement the API
The first thing we need to do is agree on how the API will be consumed. My proposal is:
{% highlight javascript %}
function Vehicle(hasEngine, hasWheels) {
    this.hasEngine = hasEngine || false;
    this.hasWheels = hasWheels || false;
};

function Car (make, model, hp) {
    this.hp = hp;
    this.make = make;
    this.model = model;
};

Car.prototype = new Vehicle(true, true);

var car = immutable(new Car('Hyundai', 'Elantra', 10));
var car2 = car
  .withHp(car.hp + 20) // creating with an existing property and a new value
  .withHasMirrors(true); // creating with a new property;

console.log(car2.hp); // prints 30
console.log(car2.hasMirrors); // prints true
{% endhighlight %}

Now let's set some requirements:

* When calling `immutable(obj)` the object `obj` becomes immutable.
* The `with...` methods only perform a shallow copy. They should maintain the prototype chain.
* Only _fields_ can be set using `with...`, methods can't.

> Before we get into the implementation details it's important to remark that the [Proxy feature](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) is part of the [unfinished ECMAScript 6 specification](https://developer.mozilla.org/en-US/docs/Web/JavaScript/ECMAScript_6_support_in_Mozilla). The [documentation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy) is really good so I won't delve into the details of how the feature works.

A potential implementation could look like this:
{% highlight javascript %}
var immutable = (function(){
  function isFunction(f) {
    var getType = {};
    return f && getType.toString.call(f) === '[object Function]';
  }

  function shallowCopy(object){
    var copy = Object.create(Object.getPrototypeOf(object));

    Object.getOwnPropertyNames(object).forEach(function (name){
      copy[name] = object[name];
    });

    return copy;
  }

  return function immutable(object){
    var proxy = new Proxy(object, {
      get: function(target, originalName){
        if (originalName.indexOf('with') === 0 && originalName.length > 4)
        {
          var capitalizedName = originalName.substring(4);
          var name = capitalizedName.charAt(0).toLowerCase() + capitalizedName.slice(1);
          var attribute = target[name];
          if (!attribute || !isFunction(attribute)){
            return function(value){
              var copy = shallowCopy(object);
              copy[name] = value;
              return immutable(copy);
            };
          }
        }

        return target[originalName];
      }
    });

    Object.freeze(object);

    return proxy;
  };
})();
{% endhighlight %}

# Testing things out
At the time of writing the [only browser that supports the latest version of the specification](http://kangax.github.io/compat-table/es6/) is Firefox. I'm using version 29.0.1 and running [this gist](https://gist.github.com/dschenkelman/99e0453d58cea7e02d62).

The output is:
{% highlight javascript %}
10 sample.js:59
"Hyundai" sample.js:60
"Elantra" sample.js:61
true sample.js:62
true sample.js:63
undefined sample.js:64
30 sample.js:68
"Hyundai" sample.js:69
"Elantra" sample.js:70
true sample.js:71
true sample.js:72
true sample.js:73
{% endhighlight %}
