# Getting Started
Welcome to Group A's ChatApp API! Here's a quick guide you can follow to get set up implementing it.

## Setting up your connections
Your first step on the path to chatting is setting up your connections so that you can receive messages from other apps!

This takes place in a couple steps:
### Create your <xref:common.connection.IConnection> implementation.
IConnection is the object that will receive all high-level, non-room specific communications from other apps. It has two methods you should override. One, `receiveMessage`, which will be called whenever another app sends you a message. Another, `setConnection` which will send you the <xref:common.connection.INamedConnection> of another party, establishing auto-connect back functionality. Here's what your implementation might look like:

```java
public class AppConnection implements IConnection {
    
    // ...

    @Override
    public void setConnection(INamedConnection remote) {
        // we received another connection, add it to our list!
        adapter.addConnection(remote);
    }

    @Override
    public void receiveMessage(ConnectionDataPacket<? extends IConnectionMessage> message) throws RemoteException {
        // we just got a message, execute it on our visitor
        message.execute(this.extendedVisitor);
    }
}
```

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
Once you receive an <xref:common.connection.IInitialConnection> stub from another machine, either via the Discovery Server or manual IP connection, save it in a list of connected apps somewhere. It will let you invite that app to a chatroom.

## Creating a chatroom
To create a chatroom, simply create a channel in the [PubSubSync](https://canvas.rice.edu/courses/57240/pages/using-the-pubsubsync-library) server. The channel's data should be of type `HashSet<INamedMessageReceiver>` and should be empty to start. The friendly name of the PubSubSync channel should be what you want to call the chatroom.

> [!TIP]
> While you might think that the roster should contain your own <xref:common.room.INamedMessageReceiver> to start, it's probably a good idea to separate that part out. Then, the logic for joining someone else's and creating your own chatroom look the same!

> [!NOTE]
> Whenever that channel's update listener is called, be sure to update your own roster in your mini-view if you display it!

That's it! With this set of <xref:common.room.INamedMessageReceiver>s, you'll be able to send messages to everyone in the chatroom. But more on that later. For now, we need to get some people in the chatroom!

## Inviting someone to a chatroom
With this chatroom created, the PubSubSync server will give you a unique channel ID that you can use to identify the chatroom. That ID and the name is all you need to invite someone else to a chatroom. Remember that <xref:common.connection.IInitialConnection> stub we got from the remote machine earlier? We can use that now to send an <xref:common.messaging.connection.IInviteMessage>. Here's an example that uses the provided <xref:common.messaging.connection.InviteMessage> convenience code:

```java
remoteNamedConnection.getConnection().receiveMessage(
    new ConnectionDataPacket<>(
        new InviteMessage(channel.getFriendlyName(), channel.getChannelId()), // the invitation itself
        this.namedConnection // the sender -- our named connection!
    )
);
```

This will typically be called by the model, when the view uses its adapter to invite someone to a room.

## Handling an Invitation
On the flip side, you also need to handle receiving an <xref:common.messaging.connection.InviteMessage>. In the visitor (an instance of <xref:common.connection.ConnectionDataPacketAlgo>) we set up earlier, you can handle the invite message. This example uses the <xref:common.messaging.connection.SimpleConnectionCmd> to help simplify code and an `adapter` to the main model, to let it process the invite as it sees fit.

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

## Joining a chatroom
Once you've been invited, you can join a chatroom! 