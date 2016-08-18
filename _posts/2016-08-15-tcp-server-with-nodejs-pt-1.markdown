---
layout: post
title:  "Tutorial: TCP Server with Node.js - [pt. 1]" 
date: 2016-08-15T00:12:30-03:00
categories: javascript node-js server programming
cover_picture: node.png # this picture is inside /img/posts/cover
author: "Richard Eitz"
---

Topics we will be doing in this series:

* Setup and configure a Node.js application
* Understand how TCP works
* Handle incoming connections and messages
* Manage our clients
* Broadcast events to clients

This series will be done in paralel with our other series "[C# - Unity3D TCP Client][unity-tcp-series]", so it's possible that we will be referencing material from there sometimes. But, if you already know how to write your own client in your favorite language, you won't need it because all the tests can be done with telnet.

----


## Setting up

For this post I am going to assume that you already have the lastest version of [Node.js][node] installed and working, and the command `node` is avaliable in your command line. You can test this with `node -v`. If the output was >= v4.2.6, you are probably safe to go ahead, if not, I recommend that you upgrade your Node.js.

### About Node

Node.js has a very neat feature that enables the possibilty of importing code from other **.js** files, this is done through the `require` function. Using this we can separate the roles of our application in different classes and files. To use an object, class, function or variable from another script in the same folder, you can use the following snippet:

{% highlight js %}
var yourClass = require('./yourClass');
{% endhighlight %}

### The TCP protocol

By [Wikipedia][tcp-wikipedia], Transmission Control Protocol:

> ... provides reliable, ordered, and error-checked delivery of a stream of octets between applications running on hosts communicating over an IP network. 

This means that the protocol can guarantee that what you send through the network and over IP is always arrived at the other end. TCP also has the special feature of keeping the connections opened for later referece and usage. An usual connection flow:

* Client connects to the server at an IP Address and a Port (Ex: 127.0.0.1:3000)
* Server accepts the connection, now the client can keep a reference to the pipe that goes to the server and the server can also do the same.
* Both send messages to each other.
* The connection can be closed by any side.


### Server.js
{% highlight js %}
// Load the TCP Library
const net = require('net');
// importing Client class
const Client = require('./Client');

const Server = function (port, address) {
  
  // Currently connected clients
  var clients = [];

  this.port = port || 5000;
  this.address = address || 'localhost';
  
  // Broadcast messages to the network
  this.broadcast = (message, clientSender) => {
    clients.forEach((client) => {
      // Sender doesn't receive it's own message
      if (clientSender != undefined && client === clientSender)
        return;
      client.receiveMessage(message);
    });
    console.log(message.replace(/\n+$/, ""));
  };

  this.start = function(callback) {

    /*
     * Creating New Server
     * The callback get's called when a new client joins the server
    */ 
    var server = this;
    this.connection = net.createServer((socket) => {
      
      var client = new Client(socket);

      // Validation, if the client is valid
      if (!validateClient(client)) {
        client.socket.destroy();
        return;
      }
      // Broadcast the new connection
      server.broadcast(client.name + " connected.\n", client);
      // Storing client for later usage
      clients.push(client);

      // Triggered on message received by this client
      socket.on('data', (data) => { 
        // Broadcasting the message
        server.broadcast(client.name + " -> " + data, client);
      });
      
      // Triggered when this client disconnects
      socket.on('end', () => {
        // Removing the client from the list
        clients.splice(clients.indexOf(client), 1);
        // Broadcasting that this player left
        server.broadcast(client.name + " disconnected.\n");
      });

    });
    // starting the server
    this.connection.listen(this.port, this.address);
    
    // setuping the callback of the start function
    if (callback != undefined) {
      this.connection.on('listening', callback);  
    }

  };

  /*
   * An example function: Validating the client
   */
  function validateClient(client)  {
    return client.isLocalhost();
  }
  
  return this;
};

module.exports = Server;
{% endhighlight %}

### Client.js
{% highlight js %}
const Client = function (socket) {
  this.address = socket.remoteAddress;
  this.port    = socket.remotePort;
  this.name    = this.address + ":" + this.port;
  this.socket  = socket;

  this.receiveMessage = (message) => {
    this.socket.write(message);
  };

  this.isLocalhost = () => {
    return this.address == "127.0.0.1" || this.address == "localhost";
  };  
};

module.exports = Client;
{% endhighlight %}

### init.js
{% highlight js %}
// importing Server class
const Server = require('./Server');

// Our configuration
const PORT = 46141;
const ADDRESS = "127.0.0.1"

var server = new Server(PORT, ADDRESS);

/* Starting our server
 * The callback parameter get's called after the
 * initializing fase is done
 */
server.start(function () {
  console.log("Server running, port: " + server.port)
});
{% endhighlight %}

[node]: https://nodejs.org/en/
[unity-tcp-series]: #
[tcp-wikipedia]: https://en.wikipedia.org/wiki/Transmission_Control_Protocol
[udp-wikipedia]: #