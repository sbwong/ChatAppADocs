Here's a simple example of how you could pass an <xref:common.room.ICmd2LocalSystemAdapter> factory to a <xref:common.room.RoomDataPacketAlgo>. Each method on the `viewAdapter` takes in a String representing the sender's name.

```java
new ChatroomMessageExtendedVisitor(new IVisitorToModelAdapter() {
    // ...
}, (sender) -> new ICmd2LocalSystemAdapter() {
    @Override
    public void writeText(INamedMessageReceiver nmr, String text) {
        viewAdapter.writeText(senderName.toString(), text);
    }
    @Override
    public void writeImage(Image image) {
        viewAdapter.writeImage(senderName.toString(), image);
    }
    @Override
    public void writeError(String error) {
        viewAdapter.writeError(senderName.toString(), error);
    }
    @Override
    public void addComponent(Supplier<JComponent> componentSupplier) {
        viewAdapter.addComponent(senderName.toString(), componentSupplier);
    }
});
```