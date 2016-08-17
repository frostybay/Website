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

{% highlight js %}
// TcpServer.js

var TcpServer = function(port, address) {
  // Load the TCP Library
  const net = require('net');
  this.port = port || 5000;
  this.address = address || 'localhost';
  
  var clients = [];

  var events = {
    'clientConnected' : function(){ return true },
    'message' : function(){},
    'clientDisconnected' : function(){},
  };

  this.on = function (event, callback) {
    if (!(event in events))
      throw new Error("This event doesn't exist: " + event);
    events[event] = callback;
    return this;
  };

  this.serverBroadcast = function (message) {
    broadcast(undefined, "SERVER: " + message + "\n");
    return this;
  }

  this.clientBroadcast = function (client, message) {
    broadcast(client, client.name + " SENT: " + message);
    return this;
  }

  function broadcast (clientSender, message) {
      clients.forEach(function (client) {
        // Don't want to send it to sender
        if (client === clientSender) return;
        client.send(message);
      });
    };    

  this.connection = net.createServer(function(socket) {

    var client = new Client(socket);
    // validate
    if (!events['clientConnected'](client)) {
      client.socket.destroy();
      return;
    }

    // new client!
    clients.push(client);

    // setup events
    socket.on('data', function(data) { 
      events['message'](client, data);
    });
    socket.on('end', function () {
      clients.splice(clients.indexOf(client), 1);
        events['clientDisconnected'](client);
    });

  }).listen(this.port, this.address);

  return this;
};

// Client.js

var Client = function (socket) {
  this.address = socket.remoteAddress;
  this.port    = socket.remotePort;
  this.name    = this.address + ":" + this.port;
  this.socket  = socket;

  this.send = function(message) {
    this.socket.write(message);
  };

  this.isLocalhost = function () {
    return this.address == "127.0.0.1" || this.address == "localhost";
  }
};

// main.js

var server = new TcpServer(46131, "127.0.0.1");

server
  .on("clientConnected", function(client) {
    if (client.isLocalhost()) {
      server.serverBroadcast(client.name + " connected.");
      return true;
    } else {
      return false; 
    }   
  })
  .on("message", function(client, message) {
    server.clientBroadcast(client, message);
  })
  .on("clientDisconnected", function (client) {
    server.serverBroadcast(client.name + " disconnected.");
  });

server.connection.on('listening', function() {
  console.log("Chat server running at port " + server.port + "\n");
});
{% endhighlight %}

[node]: https://nodejs.org/en/
[unity-tcp-series]: #