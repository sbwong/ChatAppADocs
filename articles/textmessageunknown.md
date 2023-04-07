# ITextMessage as an Unknown Message
To use <xref:common.message.room.ITextMessage> as an unknown message type, define a method on your visitor that you'll use to add unknown commands, e.g. `addNewCommand(IDataPacketID id, ARoomMessageAlgoCmd<?> cmd)`. This will be implemented like in the [Getting Started Guide](./getstarted.md#receiving-a-message-command) to instantiate an <xref:common.room.ICmd2LocalSystemAdapter>. Then, simply delegate to it for both <xref:common.message.room.ITextMessage> and when receiving commands via <xref:common.message.room.ICommandMessage>. This way the `adapter` provided to <xref:common.room.SimpleRoomCmd> is guaranteed to be non-null.

```java
// add the text message command via addNewCommand
addNewCommand(ITextMessage.ID, new SimpleRoomCmd<ITextMessage>() {
    @Override
    public void execute(ITextMessage data, INamedMessageReceiver sender, ICmd2LocalSystemAdapter adapter) {
        adapter.writeText(data.getText());
    }
});

// add unknown message commands whenever we get them
this.setCmd(ICommandMessage.ID, new SimpleRoomCmd<ICommandMessage>() {
    @Override
    public void execute(ICommandMessage data, INamedMessageReceiver sender, ICmd2LocalSystemAdapter adapter) {
        addNewCommand(data.getPacketID(), data.getMessageCommand());
    }
});
```