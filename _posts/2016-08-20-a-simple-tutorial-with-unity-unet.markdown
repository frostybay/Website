---
layout: post
title:  "Tutorial: A simple chat with Unity UNET" 
date: 2016-08-20T00:12:30-03:00
categories: c# unet unity server client chat programming
author: "Carlos Cadori"
---

### Description

This tutorial will teach you how to build a simple online chat using Unity UNET system.


### Requirements

* Basic knowledge of Unity Editor.
* Basic Knowledge of C#.
* Unity3D 5.1 +.

---

### Connection and scenes

First, we need to create two scenes, online and offline scene. 

After that, drag the scenes to **BuildSettings** window (File->Build Settings).

On the offline scene, create a new empty object and add **NetworkManager** and **NetworkManagerHUD** to it.

Drag our scenes to Offline Scene and Online Scene field on the **NetworkManager**.

Uncheck the box Auto Spawn (Spawn Info -> Auto Spawn) on the **NetworkManager**.


### Connecting

Start the offline scene and press the button called “LANHost” or “LANClient” on top of the screen.

If you want to connect with *Unity Matchmaking*, you need to change some settings before. 


### Basic interface elements

To start coding, we will need some UI elements to our chat.

* Text – We show messages here.
* Input Field – We will write here.
* Button – To send the messages.

They are basic UI elements. You can create them with left-click on hierarchy window.

You can also download my prefab for this tutorial [here][chat-prefab].


### Interface methods

Create a new script to our chat.

{% highlight c# %}

using UnityEngine;
using UnityEngine.UI;

public class Chat : MonoBehaviour
{
  
}

{% endhighlight %}

We need import **UnityEngine.UI** to get access to UI classes.

Let’s create some properties to our interface elements.

{% highlight c# %}

...

  [SerializeField]
  private Text container;
  
  [SerializeField]
  private ScrollRect rect;

...

{% endhighlight %}

>The “SerializeField” tag force property serialization.

Now, we need a method to add messages to our chat. Let’s create with internal access, because in the future we will override it.

{% highlight c# %}

...

  internal void AddMessage(string message)
  {
    container.text += "\n" + message;
  }

...

{% endhighlight %}

We will need also a method to send messages to the server. This method will be just a trigger to our extended class.

{% highlight c# %}

...

  public virtual void SendMessage(InputField input) {}

...

{% endhighlight %}

Attach this method to the event “OnClick” on the button that we have created before.


### The client

Create a new script to extend our **Chat** class.

Do not forget to import some namespaces to use the *UNET*.

{% highlight c# %}

using UnityEngine;
using UnityEngine.Networking;
using UnityEngine.Networking.NetworkSystem;

public class UNETChat : Chat
{

}

{% endhighlight %}

We will define a code to our chat messages.

{% highlight c# %}

...

  //just a random number
  private const short chatMessage = 131;

...

{% endhighlight %}

Now, we will create a method to receive our messages. We will receive a **NetworkMessage** object as a parameter and send to a previously created method on **Chat**.

{% highlight c# %}

...
  
  private void ReceiveMessage(NetworkMessage message)
  {
    //reading message
    string text = message.ReadMessage<StringMessage> ().value;

    AddMessage (text);
  }

...

{% endhighlight %}

On start method (called by unity whenever the scene start), we will register a handle to our client, who can be accessed by the **NetworkManager** singleton (I can explain singletons in another post). It means that whenever we receive a message with 131 code, we send to the method *ReceiveMessage*.

{% highlight c# %}

...

  private void Start()
  {
    //registering the client handler
    NetworkManager.singleton.client.RegisterHandler (chatMessage, ReceiveMessage);
  }

...

{% endhighlight %}

To send our messages, we need to override the method *SendMessage*.

{% highlight c# %}

...

  public override void SendMessage (UnityEngine.UI.InputField input)
  {
    StringMessage myMessage = new StringMessage ();
    //getting the value of the input
    myMessage.value = input.text;

    //sending to server
    NetworkManager.singleton.client.Send (chatMessage, myMessage);
  }

...

{% endhighlight %}

Let’s get the text inside the input field and set as value of a new instance of **StringMessage**.

>If you want to put more information in the message, you can also create your own message class extending Message Base.

Using the method *SendMessage* of our client, we send the message with the identifier code.

### The server

Here, I will make server and client in the same script, but you can separate them if you want. 

Let’s create a new method to receive client messages in the server and send to other clients.

{% highlight c# %}

...

  private void ServerReceiveMessage(NetworkMessage message)
  {
    StringMessage myMessage = new StringMessage ();
    //we are using the connectionId as player name only to exemplify
    myMessage.value = message.conn.connectionId + ": " + message.ReadMessage<StringMessage> ().value;

    //sending to all connected clients
    NetworkServer.SendToAll (chatMessage, myMessage);
  }

...

{% endhighlight %}

The method creates a new **StringMessage** and set the received string as value, using connection id of the sender client as player name.

We are using the connection id to keep this tutorial simple, but if you want to use nicknames, just send with a custom **NetworkMessage** or create a reference in the server.

At the end, we just send to all clients with **NetworkServer** (You only can use this class on the server).

*Relax. It is almost ready…*

Now, we have to add some lines on our *Start* method.

{% highlight c# %}

...
  ...

    //if the client is also the server
    if (NetworkServer.active) 
    {
      //registering the server handler
      NetworkServer.RegisterHandler(chatMessage, ServerReceiveMessage);
    }

  ...
...

{% endhighlight %}

First, let’s check if this instance is running on server (or host). After that, we just register a new server handler.

Well, it is working. You can test building your project and opening another application.


### Considerations

I hope this tutorial had helped you.

If you have some question, or found a bug, please, comment below! :D

Full code [here][repo]!


### References

[Unity NetworkServer][server]

[Unity NetworkClient][client]

[Unity MessageBase][message]

[Unity InputField][input]

[chat-prefab]: https://github.com/frostybay/tutorial-chat/tree/master/Assets/Prefabs
[repo]: https://github.com/frostybay/tutorial-chat
[client]: https://docs.unity3d.com/ScriptReference/Networking.NetworkClient.html
[server]: https://docs.unity3d.com/ScriptReference/Networking.NetworkServer.html
[input]: https://docs.unity3d.com/Manual/script-InputField.html
[message]: https://docs.unity3d.com/ScriptReference/Networking.MessageBase.html