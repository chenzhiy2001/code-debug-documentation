# code-debug 调试器原理

在本章中，我会简要地说明调试器的工作原理，方便您快速上手调试。

由于大部分调试器的用户都在Qemu虚拟机上进行OS开发，我们先来讨论调试Qemu上的OS的情况。

Qemu提供了GDBServer，我们让一个中间人（叫做Debug Adapter）和这个GDBServer交互（本插件的大部分代码都是Debug Adapter），Debug Adapter再和VSCode交互（这部分代码在src/frontend/extension.ts·里）:

```
VSCode -- Debug Adapter -- Qemu GdbServer
```

您在使用本插件进行OS调试时，可以在`Debug Console`中看到它们之间的消息传送。Json格式的消息是Debug Adapter传给VSCode的，文本格式的消息（例如`14-exec-next thread 1`）是Debug Adapter传给Qemu GDBServer的。