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

<!-- 
  This series will be done in paralel with our other series "[C# - Unity3D TCP Client][unity-tcp-series]", so it's possible that we will be referencing material from there sometimes. But, if you already know how to write your own client in your favorite language, you won't need it because all the tests can be done with telnet.
--> 

All the client tests will be done with [telnet][telnet] software. There is not a client source in this tutorial.

----


## Good-to-know things:

### About Node

For this post I am going to assume that you already have the lastest version of [Node.js][node] installed and working, and the command `node` is avaliable in your command line. You can test this with `node -v`. If the output was >= v4.2.6, you are probably safe to go ahead, if not, I recommend that you upgrade your Node.js.

Node.js has a very neat feature that enables the possibilty of importing code from other **.js** files, this is done through the `require` function. Using this we can separate the roles of our application in different classes and files. To use an object, class, function or variable from another script in the same folder, you can use the following snippet:

{% highlight js %}
var yourClass = require('./yourClass');
{% endhighlight %}

### About ES6

I assume a basic knowledge of javascript (ES6) as in this tutorial you will see the usage of three new features of javascript. This first one is the [class][js-class]. The second one is [arrow functions][js-arrow_functions]. And the third one are the quotes like these: \`string\`. Inside them we can interpolate variables using this notation: `Stringy and ${var} are friends`. They are just syntatic sugar for writing better and cleaner code.

Examples:

{% highlight js %}

// Class & String interpolation
class Person {
  constructor (name) {
    this.name = name;
  }
  greet (otherPerson) {
    console.log(`${this.name} says: hello ${otherPerson.name}!`);
  }
}
var a = new Person('Bob');
var b = new Person('Dude');
a.greet(b); // Bob says: hello Dude!

// Arrow function
executeSomething((firstParam, secondParam) => {
  // callback_code
});
{% endhighlight %}

### The TCP protocol

A little introduction to the TCP protocol:

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

{% highlight js %}
#!/usr/bin/env node
'use strict';

// load the Node.js TCP library
const net = require('net');

const PORT = 5000;
const ADDRESS = '127.0.0.1';

let server = net.createServer(onClientConnected);
server.listen(PORT, ADDRESS);

function onClientConnected(socket) {
  console.log(`New client: ${socket.remoteAddress}:${socket.remotePort}`);
  socket.destroy();
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

{% highlight bash %}
$ chmod +x init.js
{% endhighlight %}

Then, start it up with:

{% highlight bash %}
$ ./init.js
{% endhighlight %}

By now you should've got the message *Server started at: 127.0.0.1:5000*. 

This means that we are ready to test it using a simple but powerful tool: [telnet][telnet]. With telnet we can connect to TCP server very easily, just typing `telnet HOST PORT` and we are connected.

Let's do it. Fire up another terminal (protip: `Ctrl+Alt+T`) and run the following line: 

{% highlight bash %}
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
function onClientConnected(socket) {

  // Giving a name to this client
  let clientName = `${socket.remoteAddress}:${socket.remotePort}`;

  // Logging the message on the server
  console.log(`${clientName} connected.`);

  // Triggered on data received by this client
  socket.on('data', (data) => { 

    // getting the string message and also trimming
    // new line characters [\r or \n]
    let m = data.toString().replace(/[\n\r]*$/, '');

    // Logging the message on the server
    console.log(`${clientName} said: ${m}`);

    // notifing the client
    socket.write(`We got your message (${m}). Thanks!\n`);
  });
  
  // Triggered when this client disconnects
  socket.on('end', () => {
    // Logging this message on the server
    console.log(`${clientName} disconnected.`);
  });
}
{% endhighlight %}

The code itself is pretty self explanatory with the comments, but, just for sure: the function `onClientConnected` get's executed only one time, when the client connects. In it we are giving the client a name and setting up some events. The events are:

#### About on('data') & on('end')

These are both events that will be triggered (get executed) when special events happen.

**'data'**: happens when the server receives data from the client. The param `data` can be a buffer of bytes or just a string. We'll be using it as a string for now.

Inside of the callback we're logging the message to the console of the server and also notifying the client that the message was succesfully recieved.

**'end'**: it happens when the client disconnects from the server. For now we are just logging who disconnected in the server's console.


**Time to test!** You can try again using the telnet client:

{% highlight bash %}
$ telnet 127.0.0.1 5000
{% endhighlight %}

This time instead of a instant disconnect we get a prompt for writing something. 

Go ahead and write a message for the server! You can keep both terminals open so you can see it happen in action.

*Tip: Now should be a good time to say that to "get out" of the telnet client you should press* `Ctrl+]` *and then* `q`*. This is what the escape character was all about.*

Quit telnet now.

If everything again went well, the server output should be like this after we've done that.

**server**

{% highlight bash %}
$ ./init.js 
Server started at: 127.0.0.1:5000
127.0.0.1:54052 connected.
127.0.0.1:54052 said: Hey
127.0.0.1:54052 disconnected.
{% endhighlight %}

And the client:

**client**

{% highlight bash %}
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

It's working! The client's port and the message you typed maybe different, but that's cool.

I guess it is time to make this a real server enabling multiple connections (clients) that can speak to each other. Let's update our code once again.

### Multiple clients, multiple fun

Things are getting big now, it's only a matter of time for the messy code to arrive.

**Let's be cautious**: It is time to start separating the roles to different classes and files. Our server will now live inside Server.js, our client will have it's own class and file too: Client.js. We will keep the init.js, it will still be our application that glues everything.

Our init.js should be upgraded again. It is now very neat and simple:

**init.js**

{% highlight js %}
#!/usr/bin/env node
'use strict';

// importing Server class
const Server = require('./Server');

// Our configuration
const PORT = 5000;
const ADDRESS = "127.0.0.1"

var server = new Server(PORT, ADDRESS);

// Starting our server
server.start(() => {
  console.log(`Server started at: ${ADDRESS}:${PORT}`);
});
{% endhighlight %}

Ok, we'll be moving all the server logic to a file name Server.js. You should copy the contents below. It's a big file, so you can take your time to try to understand by yourself what's going on or wait a little bit that everything is going to get explained.

We moved the function `onClientConnected` to the inside of the `start` function of the server. It's a good place to start looking.

*If you look carefully over the Server.js you may notice some TODOS. These are work that we'll be doing in the next part.*

**Server.js**

{% highlight js %}
'use strict';
const net = require('net');
const Client = require('./Client'); // importing Client class

class Server {
  constructor (port, address) {
    this.port = port || 5000;
    this.address = address || '127.0.0.1';
    // Holds our currently connected clients
    this.clients = [];
  }
  /*
   * Starting the server
   * The callback argument is executed when the server finally inits
  */ 
  start (callback) {
    let server = this; // we'll use 'this' inside the callback below
    // our old onClientConnected
    server.connection = net.createServer((socket) => { 
      let client = new Client(socket);
      console.log(`${client.name} connected.`);

      // TODO 1: Broadcast to everyone connected the new client connection

      // Storing client for later usage
      server.clients.push(client);
      
      // Triggered on message received by this client
      socket.on('data', (data) => { 
        let m = data.toString().replace(/[\n\r]*$/, '');
        console.log(`${clientName} said: ${m}`);
        socket.write(`We got your message (${m}). Thanks!\n`);
        // TODO 2: Broadcasting the client's message to the other clients
      });
      
      // Triggered when this client disconnects
      socket.on('end', () => {
        // Removing the client from the list
        server.clients.splice(server.clients.indexOf(client), 1);
        console.log(`${client.name} disconnected.`);
        // TODO 3: Broadcasting that this client left
      });
    });
    // starting the server
    this.connection.listen(this.port, this.address);
    // setuping the callback of the start function
    this.connection.on('listening', callback);
  }

  // TODO
  broadcast(message, clientSender) {}
}
module.exports = Server;
{% endhighlight %}

#### Understanding Server.js before executing again

Let's review all the new methods of Server class:

{% highlight js %}
constructor (port, address)
{% endhighlight %}

Our Server class constructor isn't doing much, just setting up everything that's going to be needed for the start function. It expects a port and an address. A new thing here is that we've initialized the `clients` array. It will hold our clients for later usage.

{% highlight js %}
start (callback)
{% endhighlight %}

We've copied the contents of onClientConnected here and made some modifications for it to work inside the class. We are also instantiating Client instances. The callback received is executed when the server finishes it's initializing fase. Other than that it's almost the same code we had before, with the exception that we've added some TODOS to be done. These are addressed below:

{% highlight js %}
// TODO
broadcast(message, clientSender)
{% endhighlight %}

As we didn't knew anything about broadcasting, before adding any code I should explain what it is:

The idea of broadcasting information is like almost exactly like the everyday usage of the word for radios or TV. **Broadcasting is the act of sending a message to everyone on the network.**

Let's continue to finish the server before executing it, it's time to update your broadcast function and here's how we gonna do it:

{% highlight js %}
/*
 * Broadcasts messages to the network
 * The clientSender doesn't receive it's own message
*/
broadcast (message, clientSender) {
  this.clients.forEach((client) => {
    if (client === clientSender)
      return;
    client.receiveMessage(message);
  });
  console.log(message.replace(/\n+$/, ""));
}
{% endhighlight %}

An iteration on the clients array (They get stored as they connect to the server). And a simple call to the client's method `receiveMessage` to send the message. We're also logging in the server all the messages sent, without the trailling new line.

As we didn't had a broadcast function before, we couldn't contact our clients when something happened on the network. But now he have one! Let's update our previous TODOs as well.

*\* TODO 1: Broadcast to everyone connected the new client connection*
  
We are going to easily just update the line with the following code:

{% highlight js %}
// Broadcast the new connection
server.broadcast(`${client.name} connected.\n`, client);
{% endhighlight %}

The broadcast receives two parameters, one is the message itself and the other is the client who emmited the message. He shouldn't receive his own message.

*\* TODO 2: Broadcasting the client's message to the other clients*

Now that we know how to use the broadcast function we can just update the the TODO 2 as well. You can replace everything inside the 'data' event because the logging is already being done inside the broadcast method.

{% highlight js %}
// Triggered on message received by this client
socket.on('data', (data) => { 
  // Broadcasting the message
  server.broadcast(`${client.name} says: ${data}`, client);
});
{% endhighlight %}

*\*TODO 3: Broadcasting that this client left*

Again, just a simple usage of the broadcast function. Update the whole 'end' event:

{% highlight js %}
// Triggered when this client disconnects
socket.on('end', () => {
  // Removing the client from the list
  server.clients.splice(server.clients.indexOf(client), 1);
  // Broadcasting that this player left
  server.broadcast(`${client.name} disconnected.\n`);
});
{% endhighlight %}

And finally: our Client class, super simple. Just copy the contents to a new file called Client.js:

**Client.js**

{% highlight js %}
'use strict';

class Client {
  
  constructor (socket) {
    this.address = socket.remoteAddress;
    this.port    = socket.remotePort;
    this.name    = `${this.address}:${this.port}`;
    this.socket  = socket;
  }

  receiveMessage (message) {
    this.socket.write(message);
  }

}
module.exports = Client;
{% endhighlight %}


All the complete source for the three files can be found here: [Server.js][Server.js], [Client.js][Client.js] and [init.js][init.js].

## Finalizing and the final test

If you followed the tutorial copying everything in the right place we should be ready to use the server to it's full potential. Go ahead and execute the `init.js` program, aditionally open two new terminals for the clients. In both of them use the telnet as we did before to connect, send messages and disconnect/reconnect.

The messages should be flowing from client to server and then from server to all the other clients. The server is also logging the conversation for stalking purposes (:P).

## The end & The full source code

This is it, if you have any questions that were not answered in the tutorial above, feel free to ask in the comments section.

The final source code can be also be found [here][repository].

[repository]: https://github.com/frostybay/tutorial-nodejs-tcp-server
[node]: https://nodejs.org/en/
[unity-tcp-series]: #
[tcp-wikipedia]: https://en.wikipedia.org/wiki/Transmission_Control_Protocol
[js-strict]: https://developer.mozilla.org/pt-BR/docs/Web/JavaScript/Reference/Strict_mode
[js-class]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes
[js-arrow_functions]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/Arrow_functions
[telnet]: https://en.wikipedia.org/wiki/Telnet
[Server.js]: https://github.com/frostybay/tutorial-nodejs-tcp-server/blob/master/Server.js
[Client.js]: https://github.com/frostybay/tutorial-nodejs-tcp-server/blob/master/Client.js
[init.js]: https://github.com/frostybay/tutorial-nodejs-tcp-server/blob/master/init.js