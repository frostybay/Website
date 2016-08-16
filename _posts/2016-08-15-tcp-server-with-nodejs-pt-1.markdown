---
layout: post
title:  "JS - TCP Server with Node.js - [pt. 1]" 
date: 2016-08-15T00:12:30-03:00
categories: javascript node-js server programming
cover_picture: node.png # this picture is inside /img/posts/cover
author: "Richard Eitz"
---

Topics we will try to do in this series:

* Setup and configure a Node.js application
* Understand how TCP works
* Handle incoming connections and messages
* Manage our clients
* Broadcast events to clients

This series will be done in paralel with our other series "[C# - Unity3D TCP Client][unity-tcp-series]", so we will be referencing material from there many times. But, if you already know how to write your own client in your favorite language, you won't need it.

----


## Setting up

For this post I am going to assume that you already have the lastest version of [Node.js][node] installed and working, and the command `node` is avaliable in your command line. You can test this with `node -v`. If the output was >= v4.2.6, you are probably safe to go ahead, if not, I recommend that you upgrade your Node.js.


### to be continued...

<!--
{% highlight js %}
print_this("Hello World");

function print_this(txt) {
  console.log(txt);
}
//=> prints 'Hello World' to console.
{% endhighlight %}
-->

[node]: https://nodejs.org/en/
[unity-tcp-series]: #