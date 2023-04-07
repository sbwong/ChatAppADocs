# SquareMessage
A sample of an unknown message type that squares a number!

This process takes two steps. First, we have to define the message type `SquareMessage`. This involves implementing the `IRoomMessage` interface:

```java
public class SquareMessage implements IRoomMessage {
    public static IDataPacketID ID = DataPacketIDFactory.Singleton.makeID(SquareMessage.class);

    private int n;

    public SquareMessage(int n) {
        this.n = n;
    }

    @Override
    public IDataPacketID getID() {
        return SquareMessage.ID;
    }

    public int getNumber() {
        return this.n;
    }
}
```

Now, we need a command to execute the message! This sample extends the convenience class `SimpleRoomCmd`:

```java
public class SquareMessageCommand extends SimpleRoomCmd<SquareMessage> {
    @Serial
    private static final long serialVersionUID = /* ... */;

    @Override
    public void execute(SquareMessage data, INamedMessageReceiver sender, ICmd2LocalSystemAdapter adapter) {
        int n = data.getNumber();
        adapter.writeText(sender, "Squaring " + n);
        adapter.writeText(sender, "Result: " + (n * n));
    }
}
```

Great! That's all it takes! Now all we have to do is send it!

```java
messageReceiver.receiveMessage(new RoomDataPacket<>(new SquareMessage(5), this.messageReceiver));
```

The remote app will then request our `SquareMessageCommand` and we'll provide it back!