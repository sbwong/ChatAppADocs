# Getting Started
Welcome to Group A's ChatApp API! Here's a quick guide you can follow to get set up implementing it.

## Setting up your connections
Your first step on the path to chatting is setting up your connections so that you can receive messages from other apps!

This takes place in a couple steps:
### Create your <xref:common.connection.IConnection> implementation.
IConnection is, at the end of the day, the object that will receive all communcations from other apps. It has two methods you should override. One, `receiveMessage`, which will be called whenever another app sends you a message. Another, `setConnection` which will send you the <xref:common.connection.INamedConnection> of another party, establishing auto-connect back functionality. Here's what your implementation might look like:

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
> INamedConnection **must** be `Serializable`! Make sure you are not defining it in an anonymous inner class, and that it can be properly transmitted over the network. This includes first exporting your <xref:common.connection.IConnection> as an RMI stub


## Connecting to another instance
