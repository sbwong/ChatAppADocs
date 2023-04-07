# Getting Started
Welcome to Group A's ChatApp API! Here's a quick guide you can follow to get set up implementing it.

## Setting up your connections
Your first step on the path to chatting is setting up your connections so that you can receive messages from other apps!

This takes place in a couple steps:
### Create your <xref:common.connection.IConnection> implementation.
IConnection is the object that will receive all high-level, non-room specific communications from other apps. It has a single method you should override, `receiveMessage`, which will be called whenever another app sends you a message. Here's what your implementation might look like:

```java
public class AppConnection implements IConnection {
    // ...

    @Override
    public void receiveMessage(ConnectionDataPacket<? extends IConnectionMessage> message) throws RemoteException {
        // we just got a message, execute it on our visitor
        message.execute(this.extendedVisitor);
    }
}
```

> [!WARNING]
> This guide simply uses `message.execute` for demonstration purposes. You should spin up a new thread/process the message asynchronously!

### Creating the Named Connection
So that other apps can find your name easily, you have to wrap your <xref:common.connection.IConnection> in an <xref:common.connection.INamedConnection>. This is a simple `String`, `IConnection` dyad. However, the `IConnection` this `INamedConnection` holds must be an RMI stub obtained from your IConnection instance above.

> [!NOTE]
> INamedConnection **must** be properly `Serializable`! Make sure you are not defining it in an anonymous inner class, and that it can be properly transmitted over the network. This includes first exporting your <xref:common.connection.IConnection> as an RMI stub

### Creating the Initial Connection Stub
Once you create your named connection, you have to expose it over the network so that others can connect to it. The API accomplishes this task through the use of the <xref:common.connection.IInitialConnection> interface. Since it is a functional interface with one method, `getNamedConnection`, you can use it as such: 

```java
IInitialConnection initialConnection = () -> myNamedConnection;
```

Then, create a stub from that initialConnection, and add it to the RMI registry. Now, others can connect to your app via the provided [Discovery Server](https://canvas.rice.edu/courses/57240/pages/using-the-discovery-server-package).

## Connecting to another instance
Once you receive an <xref:common.connection.IInitialConnection> stub from another machine, either via the Discovery Server or manual IP connection, save it in a list of connected apps somewhere. It will let you invite that app to a chatroom. Next you should connect back!

## Connecting back
To connect back to an <xref:common.connection.IInitialConnection>, send it an <xref:common.message.connection.IConnectionsMessage> containing your own connection, plus every other <xref:common.connection.INamedConnection> you have. That way the remote app can develop a fully connected graph of connections.

```java
remoteNamedConnection.getConnection().receiveMessage(
    new ConnectionDataPacket<>(
        new ConnectionsMessage(myConnections), // the message
        this.namedConnection // the sender -- our named connection!
    )
);
```

> [!WARNING]
> Make sure every <xref:common.connection.INamedConnection>, including the sender is an RMI **stub** not server! Otherwise you'll get some nasty `NotSerializableException`s

## Getting connected to
Then, to handle this <xref:common.message.connection.IConnectionsMessage>, just add a command in your visitor which adds all the connections you receive to the model's set of connections.

```java
this.setCmd(IConnectionsMessage.ID, new SimpleConnectionCmd<IConnectionsMessage>() {
    @Override
    public void execute(IConnectionsMessage data, INamedConnection sender) {
        adapter.addConnections(data.getConnections(), sender);
    }
});
```

This is also a good time to provide your set of connections too. Your model should compare the received list to its own, and, if it has any connections not in the list given, will send an IConnectionsMessage with all of its connections:

```java
private void addConnections(HashSet<INamedConnection> connections, INamedConnection sender) {
    // find any connections we have that aren't the sender and the sender doesn't have
    Set<INamedConnection> addlConnections = myConnections.stream().filter(c -> !c.equals(sender) && !connections.contains(c)).collect(Collectors.toSet());

    if (addlConnections.isEmpty()) return;

    // then, provide those back to the sender
    sender.getConnection().receiveMessage(
        new ConnectionDataPacket<>(
            new ConnectionsMessage(new HashSet<>(addlConnections)),
            this.namedConnection
        )
    );
}
```

Doing this will ensure that everyone has the same set of people to chat with.

## Getting into a chatroom
To actually chat with people, you should be in a chatroom! There are two ways to get into one: either create it yourself, or get invited into it. This section covers both approaches. 

### Creating a chatroom
To create a chatroom, simply create a channel in the [PubSubSync](https://canvas.rice.edu/courses/57240/pages/using-the-pubsubsync-library) server. The channel's data should be of type `HashSet<INamedMessageReceiver>` and should be empty to start. The friendly name of the PubSubSync channel should be what you want to call the chatroom.

```java
this.pubSubSyncManager.createChannel(name, new HashSet<INamedMessageReceiver>(), onNewRoster, onQuit)
```

> [!TIP]
> While you might think that the roster should contain your own <xref:common.room.INamedMessageReceiver> to start, it's probably a good idea to separate that part out. Then, the logic for joining someone else's and creating your own chatroom look the same!

> [!NOTE]
> Whenever that channel's update listener is called, be sure to update your own roster in your mini-view if you display it!

That's it! With this set of <xref:common.room.INamedMessageReceiver>s, you'll be able to send messages to everyone in the chatroom. But more on that later. For now, we need to get some people in the chatroom!

### Inviting someone to a chatroom
With this chatroom created, the PubSubSync server will give you a unique channel ID that you can use to identify the chatroom. That ID and the name is all you need to invite someone else to a chatroom. Remember that <xref:common.connection.IInitialConnection> stub we got from the remote machine earlier? We can use that now to send an <xref:common.message.connection.IInviteMessage>. Here's an example that uses the provided <xref:common.message.connection.InviteMessage> convenience code:

```java
remoteNamedConnection.getConnection().receiveMessage(
    new ConnectionDataPacket<>(
        new InviteMessage(channel.getFriendlyName(), channel.getChannelId()), // the invitation itself
        this.namedConnection // the sender -- our named connection!
    )
);
```

This will typically be called by the model, when the view uses its adapter to invite someone to a room.

### Handling an invitation
On the flip side, you also need to handle receiving an <xref:common.message.connection.IInviteMessage>. In the visitor (an instance of <xref:common.connection.ConnectionDataPacketAlgo>) we set up earlier, you can handle the invite message. This example uses the <xref:common.connection.SimpleConnectionCmd> to help simplify code and an `adapter` to the main model, to let it process the invite as it sees fit.

```java
this.setCmd(IInviteMessage.ID, new SimpleConnectionCmd<IInviteMessage>() {
    @Override
    public void execute(IInviteMessage data, INamedConnection sender) {
        adapter.inviteUserToChatroom(
                data.getChannelID(),
                data.getChatroomName(),
                sender.toString());
    }
});
```

### Joining a chatroom
Once you've been invited, you can join that chatroom! To do so, grab the channel id `channelId` from the invite message and use it to subscribe to its [PubSubSync](https://canvas.rice.edu/courses/57240/pages/using-the-pubsubsync-library) channel like so:

```java
this.pubSubSyncManager.subscribeToUpdateChannel(channelId, onNewRoster, onQuit);
```

## Talking in a chatroom
Now that you have your `IPubSubSyncChannelUpdate<>` channel, you can use its provided `HashSet<INamedMessageReceiver>` to communicate with others. But first, you should add yourself to the roster!

### Creating the <xref:common.room.IMessageReceiver>
Each <xref:common.room.IMessageReceiver> is scoped to an individual chatroom. So, when you go about making your mini-MVC the mini-M part of that should create an <xref:common.room.IMessageReceiver>. This looks extremely similar to the `IConnection` implementation from earlier. Define a class like this one to use as your message receiver:

```java
public class ChatroomMessageReceiver implements IMessageReceiver {
    // ...

    @Override
    public void receiveMessage(RoomDataPacket<? extends IRoomMessage> data) throws RemoteException {
        data.execute(visitor);
    }
}
```

`visitor` here is an instance of <xref:common.room.RoomDataPacketAlgo>, it'll process each message for your chatroom. We'll implement that later.

### Turning it into an <xref:common.room.INamedMessageReceiver> 
It's pretty simple to make your <xref:common.room.IMessageReceiver> into an <xref:common.room.INamedMessageReceiver>, and the steps are analogous to what we did before. Start by exporting a stub of the `ChatroomMessageReceiver` you created above. Then, create an <xref:common.room.INamedMessageReceiver> from it. Remember, for it to be serializable it can't be an anonymous class!

### Adding yourself to the roster
To add yourself to the roster, just call `update` on your PubSubSync channel, like so:
```java
this.pubSubChatroomChannel.update(old -> {
    old.add(receiver);
    return old;
});
```

Here's an example putting it all together in the model
```java
this.messageReceiver = new ChatroomMessageReceiver(this.visitor);
Remote messageReceiverStub = UnicastRemoteObject.exportObject(this.messageReceiver, config.stubPort);
this.namedMessageReceiver = new ChatroomNamedReceiver((IMessageReceiver) messageReceiverStub, this.username);

this.pubSubChatroomChannel.update(old -> {
    old.add(namedMessageReceiver);
    return old;
});
```

Great! You've now joined the chatroom and are ready to receive messages through your <xref:common.room.RoomDataPacketAlgo> visitor. Speaking of...

## Handling Well-Known Messages
There are only two room-level well known message types you should handle.

### <xref:common.message.room.ITextMessage>
An <xref:common.message.room.ITextMessage> represents a simple string message.
You can send an <xref:common.message.room.ITextMessage> using the convenience class <xref:common.message.room.TextMessage>:
```java
for (IMessageReceiver target : roster) {
    target.getMessageReceiver().receiveMessage(
        new RoomDataPacket<>(
            new TextMessage("Hello World!"),
            this.namedMessageReceiver
        )
    );
}
```

Then, simply process the message on your visitor:
```java
this.setCmd(ITextMessage.ID, new ARoomMessageAlgoCmd<>() {
    @Override
    public Void apply(IDataPacketID index, RoomDataPacket<IRoomMessage> host, Void... params) {
        adapter.writeText(host.getSender(), host.getData().getText());
        return null;
    }
});
```

> [!TIP]
> Alternatively, you could pretend that `ITextMessage` was an unknown type and add it to the list of visitor commands that way. See [ITextMessage as an Unknown Message](./textmessageunknown.md) for an example.

### <xref:common.message.room.IQuitRoomMessage>
This message is sent whenever someone has left the room. Simply delegate to your model adapter for processing, e.g

```java
this.setCmd(IQuitRoomMessage.ID, new SimpleRoomCmd<IQuitRoomMessage>() {
    @Override
    public void execute(IQuitRoomMessage data, INamedMessageReceiver sender, ICmd2LocalSystemAdapter a) {
        adapter.removeUserFromRoom(sender);
    }
});
```

> [!NOTE]
> For other well-known message types, see [Status Messages](#status-messages) below.

## Unknown Messages
Your <xref:common.room.RoomDataPacketAlgo> needs a couple things to start handling unknown messages properly. 

* First, it needs a way to get an instance of <xref:common.room.ICmd2LocalSystemAdapter>. The recommended approach is to use a factory that takes in a `String` or `INamedMessageReceiver` representing who sent the message. Look [here](./sampleadapterfactory.md) for an example
* You'll also need some sort of adapter to the model that can request and send unknown commands.

### Receiving an unknown message
Every unknown message should be handled by the default case on your <xref:common.room.RoomDataPacketAlgo>. This is called whenever the visitor doesn't have specific processing for that message, i.e. it is unknown.

> [!TIP]
> Although your app may define "unknown" messages, those messages aren't unknown to your app! Make sure you add those to your visitor via `setCmd` so that your app can process its own messages! It shouldn't have to send those to itself!

This default case should:
* Add the message to a list of pending messages to be processed. It doesn't know how to handle the message yet, so it has to put it somewhere while it's waiting!
* Request the unknown message command via the adapter.

```java
setDefaultCmd(new ARoomMessageAlgoCmd<>() {
    @Override
    public Void apply(IDataPacketID index, RoomDataPacket<IRoomMessage> host, Void... params) {
        pendingMessages.add(host);
        adapter.requestUnknownCommand(index, host.getSender().getMessageReceiver());
        return null;
    }
});
```

Then, in our adapter, we'll send an <xref:common.message.room.ICommandRequestMessage> to ask the sender for that command. All we need to give it is the command ID, here's an example using the convenience class <xref:common.message.room.CommandRequestMessage>:
```java
sender.receiveMessage(
    new RoomDataPacket<>(
        new CommandRequestMessage(unknownMessageId), // the message asking for our unknown message command. 
        this.namedMessageReceiver // the sender, us!
    )
);
```

> [!IMPORTANT]
> Although this simple example just sends the message directly, it's important to remember that **anytime** you send a message, you should send it on another thread to avoid blocking the main thread!

### Sending Your Message Commands Back
Oh no! Someone has requested a message command! What do you do?! Fear not, it's pretty simple!

> [!TIP]
> If you want an example of implementing an unknown message, look [here](./unknownsample.md).

First, make sure you handle the message in your visitor. This example uses the provided convenience class <xref:common.room.SimpleRoomCmd>: 
```java
this.setCmd(ICommandRequestMessage.ID, new SimpleRoomCmd<ICommandRequestMessage>() {
    @Override
    public void execute(ICommandRequestMessage data, INamedMessageReceiver sender, ICmd2LocalSystemAdapter a) {
        adapter.sendCommandWithId(data.getMessageId(), sender.getMessageReceiver());
    }
});
```

Now, all you have to do is look up the message command you have corresponding to that ID, and send back an <xref:common.message.room.ICommandMessage>.

```java
sender.receiveMessage(
    new RoomDataPacket<>(
        new CommandMessage(messageId, command), // make sure to provide the right ID -- its the message of the command that was asked for, NOT ICommandMessage.ID
        this.namedMessageReceiver
    )
);
```


> [!NOTE]
> If a message asks for the command for a well-known message type, you should respond with an <xref:common.message.room.IRejectRoomMessage> message.

> [!TIP]
> You might consider making a big `LocalMessageStore` at the main model level that you can add and get <xref:common.room.ARoomMessageAlgoCmd>s from. Then, just pass this class to the mini-model and it's easy to lookup messages when clients ask for them!

> [!TIP]
> It's a good idea to send some popular unknown message commands your app uses as soon as you connect. That way you can "preload" them for the other users in the chatroom.

### Receiving A Message Command
Receive the <xref:common.message.room.ICommandMessage> in your visitor:
```java
this.setCmd(ICommandMessage.ID, new SimpleRoomCmd<ICommandMessage>() {
    @Override
    public void execute(ICommandMessage data, INamedMessageReceiver sender, ICmd2LocalSystemAdapter adapter) {
        addNewCommand(data.getPacketID(), data.getMessageCommand());
    }
});
```
Once someone sends you a command, it's easy to install it into your visitor. Remember our <xref:common.room.ICmd2LocalSystemAdapter> factory from before? That's where this comes in! You need to take care of two things.

#### 1. Installing the command
```java
public void addNewCommand(IDataPacketID id, ARoomMessageAlgoCmd<?> cmd) {
    this.setCmd(id, new ARoomMessageAlgoCmd<>(){
        @Override
        public Void apply(IDataPacketID index, RoomDataPacket<IRoomMessage> host, Void... params) {
            ICmd2LocalSystemAdapter cmdAdapter = cmdAdapterFactory.apply(host.getSender());
            cmd.setCmd2ModelAdpt(cmdAdapter);
            cmd.apply(index, host, params);
            return null;
        }
    });

    // ...
}
```

This part can be a little confusing, so let's break it down. It does, in essence, three things
1. Installs a command into our visitor to handle all future messages with the given ID.
2. Whenever it receives a message with that ID, we create an <xref:common.room.ICmd2LocalSystemAdapter> using the factory supplied to the visitor. It takes in the sender's <xref:common.room.INamedMessageReceiver> to identify it.
3. We pass that adapter to the unknown command, then we actually execute the command using it's `apply` method -- a simple delegation of our apply to `cmd`s apply.

> [!WARNING]
> You can run into multithreading issues here! It's a good idea to wrap the `apply` body in a `synchronized` statement locking on `cmd` so that if you get two messages at once you can process one at a time.

#### 2. Executing pending messages
After adding that message to your visitor, you should execute all pending messages that have the same `IDataPacketID`. This time they'll be processed correctly by the visitor

> [!WARNING]
> Be sure to use `.equals()` and not `==` when comparing `IDataPacketID`s, otherwise you may get unexpected results.

## Leaving a chatroom
To leave a chatroom, remove yourself from the PubSubSync roster, unsubscribe from the PubSubSync channel, and send an `IQuitRoomMessage` to the rest of the room, so they know you've left!

## Conclusion
And, that's it! You should now be chatting! If there's anything this guide misses, feel free to reach out to Team A to get it added!

# Other Notes
## Status Messages
Certain well-known status messages exist to help keep apps updated about potential issues that may arise. They are as follows:

| Message                   | Description                        | Recommended Action |
|---------------------------|------------------------------------|--------------------|
|<xref:common.message.room.IRejectRoomMessage>  | Sent when the command for a well-known message type is requested | Hardcode a visitor cmd for the message ID |
|<xref:common.message.connection.IErrorConnectionMessage>  | Sent when there was an error processing a connection-level message | Send the message again, but only so many times |
|<xref:common.message.connection.IRejectConnectionMessage> | Sent when the remote refuses to process a sent message | Give up on whatever action you were trying to perform |

## Tips and Traps
