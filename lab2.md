## Lab2

---

### Overview

在Lab0中，您实现了流控制的字节流（ByteStream）的抽象;在Lab1中，您创建了一个StreamReassembler，该流接受来自同一字节流的所有子串序列，然后将它们重新组装回原始流。

这些模块将在你的后续的TCP实现中被证明是有用的，虽然这些模块不是传输控制协议所特有的细节。但是在之后的实验我们将了解传输控制协议的一些特有的细节，在Lab2中，我们将实现TCPReceiver，这是TCP中实现处理传入字节流的部分。 TCP接收器在传入的TCP段（通过Internet承载的数据报的有效载荷）和传入的字节流之间进行转换。

这是上一个实验的示意图。 TCPReceiver接收来自Internet的段（通过segment_received（）方法），并将其转换为对StreamReassembler的调用，最终将其写入传入的ByteStream。 应用程序从ByteStream读取，就像在Lab 0中通过从TCPSocket读取一样。

![](img-lab2\lab2-img-1.png)

除了写入传入流外，TCP接收器还负责告知发送方两件事：

1. “第一个未汇编”字节的索引，称为“确认号”或“ ackno”。 这是接收方需要发送方发送的第一个字节。
2. “第一个未组装的”索引和“第一个不可接受的”索引之间的距离。这称为“窗口大小”。

**ackno**和**window size**一起描述了接收者的窗口：允许TCP发送者发送的一系列索引。 接收者可以使用该窗口控制传入数据的流，使发送者限制发送的数据量，直到接收者准备好接收更多数据为止。 有时我们将ackno称为窗口的“左边缘”（TCPReceiveris感兴趣的最小索引），将ackno +窗口大小称为“右边缘”（仅超出TCPReceiveris感兴趣的最大索引）。

在编写StreamReassembler和ByteStream时，我们已经完成了实现TCPReceiver的大部分算法工作； 本Lab是关于将这些通用类连接到TCP的详细信息。 最困难的部分将涉及考虑TCP如何表示流中每个字节的位置（称为“序列号（sequence number）”）。

---

### Getting started

按照文档操作即可。

---

### Lab 2: The TCP Receiver



