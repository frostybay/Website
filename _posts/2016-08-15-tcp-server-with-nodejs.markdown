---
layout: post
title:  "Tutorial: TCP Server with Node.js" 
date: 2016-08-15T00:12:30-03:00
categories: javascript node-js server programming
cover_picture: node.png # this picture is inside /img/posts/cover
author: "Richard Eitz"
---

> **TL;DR:** This is a tutorial, so our code will be done in small parts. In case you are in a hurry and just need the final code, we've made a github repository for you: [here][repository].

Topics we will be doing in this tutorial:

* Setup and configure a Node.js application
* Understand how TCP works
* Handle incoming connections and messages
* Manage our clients
* Broadcast events to clients

This series will be done in paralel with our other series "[C# - Unity3D TCP Client][unity-tcp-series]", so it's possible that we will be referencing material from there sometimes. But, if you already know how to write your own client in your favorite language, you won't need it because all the tests can be done with telnet.

----


## Good-to-know things:

### About Node

For this post I am going to assume that you already have the lastest version of [Node.js][node] installed and working, and the command `node` is avaliable in your command line. You can test this with `node -v`. If the output was >= v4.2.6, you are probably safe to go ahead, if not, I recommend that you upgrade your Node.js.

Node.js has a very neat feature that enables the possibilty of importing code from other **.js** files, this is done through the `require` function. Using this we can separate the roles of our application in different classes and files. To use an object, class, function or variable from another script in the same folder, you can use the following snippet:

{% highlight js %}
var yourClass = require('./yourClass');
{% endhighlight %}

### About ES6

I expect a basic knowledge of javascript (ES6) as in this tutorial you will see the usage of two new features of javascript. This first one is the [class][js-class], which will be explained better later. The second one is [arrow functions][js-arrow_functions], which are just syntatic sugar for writing anonymous functions, so:

{% highlight js %}

// we will be writing our callbacks like this:
execute_something((first_param, second_param) => {
  // callback_code
});

// instead of this:
execute_something(function (first_param, second_param) {
  // callback_code
});

{% endhighlight %}

### The TCP protocol

By [Wikipedia][tcp-wikipedia], Transmission Control Protocol:

> ... provides reliable, ordered, and error-checked delivery of a stream of octets between applications running on hosts communicating over an IP network. 

This means that the protocol can guarantee that what you send through the network and over IP is always arrived at the other end. TCP also has the special feature of keeping the connections opened for later referece and usage. An usual connection flow happens like this:

* Client connects to the server at an IP Address and a Port (Ex: 127.0.0.1:3000)
* Server accepts the connection, now the client can keep a reference to the "pipe" that goes to the server and the server can also do the same.
* Both send messages to each other.
* The connection can be closed by any side.

**Enough said, let's get started!**

----

### Booting up

Use the following snippet to start our coding, copy the contents and create a file name `init.js`.

This will be our **first server**:

{% highlight javascript %}

#!/usr/bin/env node
'use strict';

// load the Node.js TCP library
const net = require('net');

const PORT = 5000;
const ADDRESS = '127.0.0.1';

let server = net.createServer(onClientConnected);
server.listen(PORT, ADDRESS);

function onClientConnected(c) {
  console.log(`New client: ${c.remoteAddress}:${c.remotePort}`);
  c.destroy();
}

console.log(`Server started at: ${ADDRESS}:${PORT}`);

{% endhighlight %}

#### Whats happening here

The first line guarantees that we can run the server as an executable file, skipping the `node` command before the script name. The second line enables [strictures][js-strict] in the file. After that, we load the required TCP library from the Node.js default libs for later usage.

{% highlight js %}
const PORT = 5000;
const ADDRESS = '127.0.0.1';
{% endhighlight %}

With these lines we are setuping our default configuration: we will start the server in localhost and at the 5000 port.

Moving for next two line, we first create a new TCP server and pass as a callback the function `on_client_connected`. This function will be executed whenever a new client connects to the server. The next line we set it up to listen at the previous configuration values.

All done! Let's start the server. Run it by first setting the file to be executed with:

{% highlight bash%}
$ chmod +x init.js
{% endhighlight %}

Then, start it up with:

{% highlight bash%}
$ ./init.js
{% endhighlight %}

By now you should've got the message *Server started at: 127.0.0.1:5000*. 

This means that we are ready to test it using a simple tool: [telnet][telnet]. With telnet we can connect to TCP server very easily, just typing `telnet HOST PORT` and we are connected.

Let's do it. Fire up another terminal (protip: `Ctrl+Alt+T`) and run the following line: 

{% highlight bash%}
$ telnet 127.0.0.1 5000
{% endhighlight %}

Ok, nice. If everything went well our telnet client tryed to connect, connected, got something about a escape character and then got a closed connection.

This is all expected.

Now go check out the terminal with our server, it did what we asked it to do: it received the connection, logged the client to the console and detroyed (closed) the pipe, Yay! A weird result, but nice.

We're ready to upgrade it.

### Upgrading our server

That's cool and all but kind of useless. The whole purpose of a network connection is to send messages from one end to another, right? 

Ok, so let's start **sending messages** from client to the server and from the server to the client. Update your function `onClientConnected` in init.js with the following code:

{% highlight js %}

function onClientConnected(c) {

  // Giving a name to this client
  let clientName = `${c.remoteAddress}:${c.remotePort}`;

  // Logging the message on the server
  console.log(`${clientName} connected.`);

  // Triggered on data received by this client
  c.on('data', (data) => { 

    // getting the string message and also trimming
    // new line characters [\r or \n]
    let m = data.toString().replace(/[\n\r]*$/, '');

    // Logging the message on the server
    console.log(`${clientName} said: ${m}`);

    // notifing the client
    c.write(`We got your message (${m}). Thanks!\n`);
  });
  
  // Triggered when this client disconnects
  c.on('end', () => {
    // Logging this message on the server
    console.log(`${clientName} disconnected.`);
  });
}

{% endhighlight %}

The code itself is pretty self explanatory with the comments, but, just for sure: the function `onClientConnected` get's executed only one time, when the client connects. In it we are giving the client a name and setting up some events. The events are:

#### On 'data'

Triggered (gets executed) when the server receives data from the client. "Data" can be a buffer of bytes or just a string. We'll be using it as a string for now.

Inside it we're logging in the server the message and also notifying the client that the message was succesfully received.

You can try again using the telnet client:

{% highlight bash%}
$ telnet 127.0.0.1 5000
{% endhighlight %}

This time instead of a instant disconnect we get a prompt for writing something. 

Go ahead and write a message for the server!

Now should be a good time to say that to "get out" of the telnet client you should press `Ctrl+]` and then `q`. This is what the escape character was all about.

Quit telnet now.

If everything again went well, the server output should be like this after we've done that.

**server**

{% highlight bash%}
$ ./init.js 
Server started at: 127.0.0.1:5000
127.0.0.1:54052 connected.
127.0.0.1:54052 said: Hey
127.0.0.1:54052 disconnected.
{% endhighlight %}

And the client:

**client**

{% highlight bash%}
$ telnet localhost 5000
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Hey
We got your message (Hey). Thanks!
^]
telnet> q
Connection closed.
{% endhighlight %}

## to be continued...

### Multiple clients, multiple fun

*Enabling multiple clients

{% highlight js %}

//  code pt 3

{% endhighlight %}

### Validation on connection

*Example of validation for client on connection

{% highlight js %}

//  code pt 4

{% endhighlight %}

## The full source code

The final code can be found [here][repository].

[repository]: https://github.com/frostybay/tutorial-nodejs-tcp-server
[node]: https://nodejs.org/en/
[unity-tcp-series]: #
[tcp-wikipedia]: https://en.wikipedia.org/wiki/Transmission_Control_Protocol
[js-strict]: #
[js-class]: #
[js-arrow_functions]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions
[telnet]: #