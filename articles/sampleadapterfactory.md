Here's a simple example of how you could pass an <xref:common.room.ICmd2LocalSystemAdapter> factory to a <xref:common.room.RoomDataPacketAlgo>. Each method on the `viewAdapter` takes in a String representing the sender's name.

```java
new ChatroomMessageExtendedVisitor(new IVisitorToModelAdapter() {
    // ...
}, (sender) -> new ICmd2LocalSystemAdapter() {
    @Override
    public void writeText(String text) {
        viewAdapter.writeText(sender.toString(), text);
    }

    @Override
    public void log(LogLevel level, String message) {
        viewAdapter.writeError(sender.toString(), level, message);
    }

    @Override
    public void addComponent(Supplier<JComponent> componentSupplier) {
        viewAdapter.addComponent(sender.toString(), componentSupplier);
    }
});
```