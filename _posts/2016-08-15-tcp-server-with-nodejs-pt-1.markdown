---
layout: post
title:  "JS - TCP Server with Node.js - [pt. 1]" 
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

This series will be done in paralel with our other series "[C# - Unity3D TCP Client][unity-tcp-series]", so we will be referencing material from there many times. But, if you already know how to write your own client in your favorite language, you won't need it.

----


## Setting up

For this post I am going to assume that you already have the lastest version of [Node.js][node] installed and working, and the command `node` is avaliable in your command line. You can test this with `node -v`. If the output was >= v4.2.6, you are probably safe to go ahead, if not, I recommend that you upgrade your Node.js.


### to be continued...

This is a sample code that you can build on while this tutorial isn't finished, in case you are here already...

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