## Lab1

---

### Overview

在Lab0中，我们使用了Internet流套接字通过Linux的传输控制协议（TCP）的内置实现从网站获取信息并发送电子邮件。 即使基础网络仅提供“尽力而为”数据传输功能，该TCP仍设法实现了产生一对可靠的有序字节流（一个从您到服务器，一个在相反方向）。 我们的指的不可靠意思是：可能丢失，重新排序，更改或复制的数据短包。 您还自己在一台计算机的内存中实现了字节流抽象。 在接下来的四个星期中，您将实现一个TCP协议，以在由不可靠的数据报网络分隔的两台计算机之间提供字节流抽象。

> *我为什么要这么做？*
>
> 在一个无差异的可靠服务之上提供一个服务或一个抽象说明了网络中许多有趣的问题。在过去的40年里，研究人员和实践者们已经发现了如何在全球范围内传递各种各样的信息——信息、电子邮件、超链接文档、搜索引擎、声音和视频、虚拟世界、协作文件共享、数字货币互联网.TCP自己的角色，使用不可靠的数据报提供一对可靠的字节流，这是一个典型的例子。有一种合理的观点认为，TCP实现是世界上使用最广泛的非平凡计算机程序。

![](D:\Note\CS144 Computer Network\img-lab1\lab1-img-1.png)

<center>图1：TCP实施中的模块排列和数据流。</center>

ByteStream是Lab0。TCP的工作是在不可靠的数据报网络上传送两个字节流（每个方向一个），以便写入连接一侧的套接字的字节以可在小便读取的字节的形式出现，反之亦然。 实验1是StreamReassembler（流量重组器），在实验2、3和4中，您将实现TCPReceiver（tcp接收器），TCPSender（tcp发送器）和TCPConnection（建立tcp连接的模块），并且最终将它们组合在一起。

Lab将要求您以模块化方式构建TCP实现。还记得您刚刚在Lab0中实现的ByteStream吗？ 在接下来的四个实验中，您最终将在网络中传送其中的两个：“向外去的（outbound）” ByteStream，用于本地应用程序将数据写入套接字并且通过TCP将数据发送到对方等待接收的端口，以及“向内来的(inbound)” ByteStream，用于等待来自对等方的数据，并且通过套接字由本地应用程序读取。 图1显示了各部分如何装配在一起。

1. 在实验1中，我们将实现一个流重组器（stream reassembler），该模块以正确的顺序将字节流的小片段（称为子字符串或片段）重组成连续且正确的字节流。
2. 在实验2中，我们将实现TCP处理接收字节流的部分：TCPReceiver。 这涉及考虑TCP如何表示流中每个字节的位置（称为“序列号”）。 TCPReceiver负责告知发送方（a）它已经成功按顺序接收多少字节流（这称为“确认”），以及（b）发送方现在允许发送多少字节（“流控制”）。
3. 在实验3中，您将实现处理出站字节流的TCP部分：TCPSender。 当发送方怀疑所传输的分段在途中丢失而从未到达接收方时，应如何应对？ 什么时候应该重试并重新传输丢失的段？
4. 在实验4中，您将把以前的工作与实验结合起来，创建一个有效的TCP实现：一个包含TCPSender和TCPReceiver的TCPConnection。您将使用它与世界各地的真实服务器进行通信。

---

### Getting started

按照文档操作即可。

---

### Putting substrings in sequence

在本Lab和下一个Lab中，您将实现一个TCP接收器：该模块接收数据报并将其转换为可靠的字节流，以供应用程序从套接字读取，就像您的Webget程序在Lab0中从Web服务器读取字节流一样。

 TCP发送方将其字节流分成若干短段（每个子串最多不超过1,460字节），以便它们每个都适合放入数据报中。 但是网络可能会对这些数据报进行重新排序，或者丢弃它们，或者不止一次传递它们。 接收者必须将这些段重新组装成它们开始时连续的字节流。

在本实验中，您将编写负责该重组的数据结构：StreamReassembler。 它将接收子字符串，该子字符串由一个字节字符串以及较大流中该字符串的第一个字节的索引组成。流中的每个字节都有自己的唯一索引，从零开始向上计数。 Stream重组器将拥有一个字节流作为输出：一旦重组器知道流的下一个字节，它将把它写入到ByteStream中。 所有者可以在需要时随时访问和读取ByteStream。