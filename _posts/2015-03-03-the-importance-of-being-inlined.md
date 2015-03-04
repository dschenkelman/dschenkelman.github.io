---
layout: post
title: 'The importance of being inlined'
categories:
- ECMAScript
- JavaScript
- v8
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
Working on [p-j-s](https://github.com/pjsteam/pjs) we found some interesting things related to the `Function` constructor and function inlining in v8.

## Context
While comparing the performance of the parallel and non-parallel implementations of `map` (docs [here](https://github.com/pjsteam/pjs#wrappedtypedarrayprototypemapmapper-done)) we found that our parallel implementation performed a lot worse than what we expected if we took into account the size of the `TypedArray` and the `mapper` function.

## The scenario
An image tranformation to apply a sepia tone effect to pictures was used for the comparison (based on [this post](http://blogs.msdn.com/b/eternalcoding/archive/2012/09/20/using-web-workers-to-improve-performance-of-image-manipulation.aspx)). The core linear implementation was:
{% highlight javascript %}
function noise() {
  return Math.random() * 0.5 + 0.5;
};

function clamp(component) {
  return Math.max(Math.min(255, component), 0);
}

function colorDistance(scale, dest, src) {
  return clamp(scale * dest + (1 - scale) * src);
};

function map(binaryData, len) {
  for (var i = 0; i < len; i += 4) {
      var r = binaryData[i];
      var g = binaryData[i + 1];
      var b = binaryData[i + 2];

      binaryData[i] = colorDistance(noise(), (r * 0.393) + (g * 0.769) + (b * 0.189), r);
      binaryData[i + 1] = colorDistance(noise(), (r * 0.349) + (g * 0.686) + (b * 0.168), g);
      binaryData[i + 2] = colorDistance(noise(), (r * 0.272) + (g * 0.534) + (b * 0.131), b);
  }
};
{% endhighlight %}

>The serial implementation works with a `Uint8ClampedArray` but accesses three elements at the same time. The parallel implementation uses a `Uint32Array` and works on one pixel (A,R,G,B) at a time.

Which was implemented as shown in the following snippet for the parallel case:
{% highlight javascript %}
pjs(new Uint32Array(canvasData.data.buffer)).map(function(pixel){
  function noise() {
      return Math.random() * 0.5 + 0.5;
  };

  function clamp(component) {
      return Math.max(Math.min(255, component), 0);
  }

  function colorDistance(scale, dest, src) {
      return clamp(scale * dest + (1 - scale) * src);
  };

  var r = pixel & 0xFF;
  var g = (pixel & 0xFF00) >> 8;
  var b = (pixel & 0xFF0000) >> 16;

  var new_r = colorDistance(noise(), (r * 0.393) + (g * 0.769) + (b * 0.189), r);
  var new_g = colorDistance(noise(), (r * 0.349) + (g * 0.686) + (b * 0.168), g);
  var new_b = colorDistance(noise(), (r * 0.272) + (g * 0.534) + (b * 0.131), b);

  return (pixel & 0xFF000000) + (new_b << 16) + (new_g << 8) + (new_r & 0xFF);
}, function(result){
  // work on result
});
{% endhighlight %}

## The problem
Taking the experience from previous benchmarks into account we considered that the performance of the two implementations with a `Uint32Array` of about 8*10^7 (eight million) elements should be similar, even favoring the parallel one. Contrary to our thoughts, the initial measurements showed that the linear implementation fared approximately four times better than the parallel one.

## The hypothesis
As the two code samples were similar, our hypothesis was that for some reason some v8 optimizations were not taking place. It was a definite possibility considering that the parallel implementation depends on the `Function`constructor to create `Function` objects in the Web Workers. Specifically we believed that the internal functions `noise`, `clamp` and `colorDistance` were not being inlined. Inlining could be the cause of both benefits and drawbacks but the belief was that performance was being affected because of lack of it.

Using [IR Hydra](http://mrale.ph/irhydra/2/) it was easy to verify the hypothesis, as the following figures show.

> The blue chevron indicates that a function has been inlined.

### Parallel implementation generated code
The figures in this section are related to the serial implementation. As it can be seen from them, all functions are being inlined.
<figure>
  <img src="{{ site.url }}/images/posts/20150303/inlined-clamp.png" alt="clamp function - serial implementation">
</figure>
<figure>
  <img src="{{ site.url }}/images/posts/20150303/inlined-colorDistance.png" alt="colorDistance function - serial implementation">
</figure>
<figure>
  <img src="{{ site.url }}/images/posts/20150303/inlined-noise.png" alt="noise function - serial implementation">
</figure>
<figure>
  <img src="{{ site.url }}/images/posts/20150303/inlined-processSepia.png" alt="processSepia function - serial implementation">
</figure>

### Parallel implementation generated code
The figures in this section are related to the parallel implementation. As it can be seen from them, only `Math.random` is being inlined.
<figure>
  <img src="{{ site.url }}/images/posts/20150303/ww-clamp.png" alt="clamp function - parallel implementation">
</figure>
<figure>
  <img src="{{ site.url }}/images/posts/20150303/ww-colorDistance.png" alt="colorDistance function - parallel implementation">
</figure>
<figure>
  <img src="{{ site.url }}/images/posts/20150303/ww-noise.png" alt="noise function - parallel implementation">
</figure>
<figure>
  <img src="{{ site.url }}/images/posts/20150303/ww-mapper.png" alt="mapper function - parallel implementation">
</figure>

## The solution
The way to solve the problem was to manually inline the functions. The resulting parallel implementation was:
{% highlight javascript %}
pjs(buff).map(function(pixel){
  var r = pixel & 0xFF;
  var g = (pixel & 0xFF00) >> 8;
  var b = (pixel & 0xFF0000) >> 16;
  var noiser = Math.random() * 0.5 + 0.5;
  var noiseg = Math.random() * 0.5 + 0.5;
  var noiseb = Math.random() * 0.5 + 0.5;

  var new_r = Math.max(Math.min(255, noiser * ((r * 0.393) + (g * 0.769) + (b * 0.189)) + (1 - noiser) * r), 0);
  var new_g = Math.max(Math.min(255, noiseg * ((r * 0.349) + (g * 0.686) + (b * 0.168)) + (1 - noiseg) * g), 0);
  var new_b = Math.max(Math.min(255, noiseb * ((r * 0.272) + (g * 0.534) + (b * 0.131)) + (1 - noiseb) * b), 0);

  return (pixel & 0xFF000000) + (new_b << 16) + (new_g << 8) + (new_r & 0xFF);
}, function(result){
  // work on result
});
{% endhighlight %}

We [created a benchmark](http://jsperf.com/pjs-map-inlining/3) to understand the difference between the initial version (the one in which functions were not inlined) and the final one (with inlining in charge of the developer).

The results for it are displayed in the following figure:
<figure>
  <img src="{{ site.url }}/images/posts/20150303/pjsMapInlining.png" alt="Comparing string encoding/decoding approaches">
</figure>
