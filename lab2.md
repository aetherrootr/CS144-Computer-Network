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

TCP是一种协议，它通过不可靠的数据报可靠地传送一对流控制的字节流（每个方向一个）。 有两个参与方参与TCP连接，并且每个参与方同时充当“发送方”（属于其自己的输出字节流）和“接收方”（属于传入的字节流）。 这两方称为连接的“端点”或“对等”。

这周，您将实现TCP的“接收器”部分，负责接收TCP段（实际的数据报有效载荷），重新组合字节流（包括字节流的结尾，发生时），并确定应将信号发送回 发送方的确认和流量控制。

> 我为什么要这样做？这些信号对于TCP在不可靠的数据报网络上提供流控制，可靠的字节流服务的能力至关重要。 在TCP中，确认表示：“接收器需要多少个下一个字节的索引，以便接收器可以重组更多的ByteStream？” 这告诉发送者它需要发送或重新发送什么字节。流控制意味着，“接收者感兴趣并愿意接收什么范围的索引？” （通常是其剩余容量的函数）。它告诉发件人可以发送多少。

#### ranslating between 64-bit indexes and 32-bit seqnos

作为热身，我们需要实现TCP表示索引的方式。 上周，我们创建了一个StreamReassembler，用于重新组装子字符串，其中每个单独的字节都有一个64位流的索引，而流中的第一个字节始终具有零索引。 64位索引足够大，我们可以将其视为永不溢出（以每秒100吉比特/秒的速度传输，要花费近50年才能达到$2^{64}$bytes。 相比之下，仅需三分之一的时间即可达到$2^{32}$bytes。）。但是，在TCP头中，空间非常宝贵，并且流中每个字节的索引不是用64位索引表示，而是用32位“序列”表示 数字”或“ seqno”。 这增加了三个复杂的地方：

1. **你需要实现一个环状的32位的编码系统。**TCP中的流可以任意长——这意味着可以通过TCP发送的ByteStream的长度没有限制。 但是$2^{32}$bytes只有4 GiB，不是很大。 所以一旦32位序列号计数到$2^{32}-1$，流中的下一个字节将具有0的序列号。
2. **TCP开始的序列号是随机值**：为了提高安全性并避免混淆属于相同端点之间较早连接的旧段，TCP尝试确保不会猜测序列号并且不太可能重复。 因此，流的序号不是从零开始。 流中的第一个序列号是称为初始序列号（ISN）的32位随机数。 这是代表SYN（流开始）的序列号，其余序列号在此之后正常运行：数据的第一个字节将具有ISN + 1的序列号（mod $2^{32}$），第二个字节将具有 ISN + 2（mod $2^{32}$）等。
3. **逻辑开始和结束分别占据一个序列号:**除了确保接收所有数据字节外，TCP还确保可靠地接收流的开始和结束。 因此，在TCP中，为SYN（流开始）和FIN（流结束）控制标志分配了序列号。 这些每个占用一个序列号。 （SYN标志占用的序列号是ISN。）流中的每个数据字节也占用一个序列号。 请记住，SYN和FIN不是流本身的一部分，也不是“字节”，它们代表字节流本身的开始和结束。

这些序列号（seqnos）在每个TCP段的头中被发送。 （而且，又有两个流，每个方向上都有一个。每个流具有单独的序列号和不同的随机ISN。）有时谈论“绝对序列号”（总是从零开始，不会换行），和大约一个“流索引”（您已经在streamReassembler中使用的内容：流中每个字节的索引，从零开始）。

了使这些区别更具体，请考虑仅包含三个字母字符串“ cat”的字节流。 如果SYN恰好具有seqno $2^{32}-2$，则每个字节的seqnos，绝对seqnos和流索引为：

| 元素    | `SYN`      | c          | a    | t    | `FIN` |
| ------- | ---------- | ---------- | ---- | ---- | ----- |
| seqno   | $2^{32}-2$ | $2^{32}-1$ | $0$  | $1$  | $2$   |
| 绝对seq | $0$        | $1$        | $2$  | $3$  | $4$   |
| 流索引  |            | $0$        | $1$  | $2$  |       |

 该图显示了TCP中涉及的三种不同类型的索引编制:

| 序列号（Sequence Number） | 绝对序列号（Absolute Sequence j） | 流索引（Stream Indices） |
| ------------------------- | --------------------------------- | ------------------------ |
| 从ISN开始                 | 从0开始                           | 从0开始                  |
| 包括SYN / FIN             | 包括SYN / FIN                     | 省略SYN / FIN            |
| 循环的32位数              | 不循环的64位数                    | 不循环的64位数           |
| “seqno”                   | “absolute seqno”                  | “stream index”           |

在绝对序列号和流索引之间进行转换很容易——只需加减一即可。 不幸的是，在序列号和绝对序列号之间进行转换会比较困难，并且将两者混淆会产生棘手的错误。 为了系统地防止这些错误，我们将使用自定义类型表示序列号：`WrappingInt32`，并编写它与绝对序列号之间的转换（以`uint64_t`表示）。`WrappingInt32`是包装类型的示例：包含内部类型的类型（在这种情况下为`uint32_t`） 但提供一组不同的功能/运算符。

我们已经为您定义了类型，并提供了一些帮助函数（请参阅wrwringintegers.hh），但是您将在wrappingintegers.cc中实现转换：

1. ```c++
   WrappingInt32 wrap(uint64t n, WrappingInt32 isn)
   ```

   `转换 absolute seqno → seqno`。给一个绝对序列号(n)和初始序列号（isn），转换为n的（相对）序列号。

2. ```c++
   uint64_t unwrap(WrappingInt32 n, WrappingInt32 isn, uint64_t checkpoint)
   ```

   `转换seqno → absolute seqno`。给定序列号（n），初始序列号（isn）和绝对检查点序列号，请计算与检查点最接近的与n对应的绝对序列号。

   注意：因为任何给定的seqno对应了许多绝对seqno，所以我们需要一个检查点来确定一个seqno来对应那一个绝对seqno。 例如，如果ISN为零，则seqno $“17”$ 对应的绝对seqno是$17$，但也对应$2^{32}+17$或$2^{33}+17$或$2^{34}+17$，等等。检查点有助于消除歧义：这是一个绝对的seqno，这个类的用户知道正确答案的“大概范围”。在这里，“大概范围”可以是指任何64位的数字是正确答案的$\pm 2^{31}$。在TCP实现中，您将使用最后一个重新组装的字节的索引作为检查点。
   
   **提示**：*最干净/最简单的实现将使用wrapping_integers.hh中提供的帮助程序功能。 包装/展开操作应保留偏移量——两个相差17的seqno将对应于两个相差17的绝对seqno。*

您可以通过运行WrappingInt32测试来测试您的实现。 在build目录中，执行`ctest -R wrap`。

#### Implementing the TCP receiver

恭喜你正确完成了wrap和unwrap逻辑！ 如果可以的话，我们可以握手。 在本实验的其余部分，您将实现TCPReceiver。 它将（1）从其对等方接收分段，（2）使用你的StreamReassembler重新组装ByteStream，（3）计算确认号（ackno）和窗口大小。 确认和窗口大小最终将在传出段中发送回对等方。

首先，请检查TCP段的格式。 这是两个端点相互发送的消息；它是较低级数据报的有效负载。 下图中非灰色的的字段表示我们在本Lab中感兴趣的信息：序列号，有效负载以及SYN和FIN标志。 这些是由发送方写入，由接收方读取并使用的字段。

![](img-lab2\lab2-img-2.png)

TCPSegment类用C ++表示此消息。 请查看TCPSegment（https://cs144.github.io/doc/lab2/classtcpsegment.html）和TCPHeader（https://cs144.github.io/doc/lab2/structtcpheader.html）的文档。 您可能对[length_in_sequencespace()](https://cs144.github.io/doc/lab2/class_t_c_p_segment.html#a41eb3ff25fee485849fd38eb31c990d6)方法很感兴趣，该方法计算一个字段占用了多少序列号（包括SYN和FIN标志每个都占用一个序列号以及有效载荷的每个字节这一事实）。

接下来，让我们讨论一下TCPReceiver将提供的接口：

```c++
// 构造一个“ TCPReceiver”，它将最多存储“ capacity”字节。
TCPReceiver(const size_t capacity); // 在.hh文件中实现

// 处理收到的TCP段
void segment_received(const TCPSegment &seg);

// ackno应发送给对等方的确认
//
// 如果未收到SYN，则返回空
//
// 这是接收者窗口的开始，换句话说，这是接收者尚未接收到的流中第一个字节的序列号。
std::optional<WrappingInt32> ackno() const;

// 窗口大小应该发送给对等方
//
// 形式上：这是接收者愿意接受的可接受索引的窗口大小。 
// 它是``第一个未组装''和``第一个不可接受''索引之间的距离。
//
// 换句话说：它是容量减去TCPReceiver在字节流中保留的字节数。
size_t window_size() const;

// 已存储但尚未重新组合的字节数
size_t unassembled_bytes() const;// 在.hh文件中实现

// 访问重组的字节流
ByteStream &stream_out();// 在.hh文件中实现
```

TCPReceiveris围绕你的StreamReassembler构建。 我们已经在.hh文件中为您实现了构造函数以及未组装的字节和流输出方法。 你接下来需要实现以下模块：

##### segment_received()

这是主要的工作方法。每次从对等方收到新的段时，都会调用`TCPReceiver::segment_received()`。

此方法需要：

- **如有必要，设置初始序列号。**具有SYN标志组第一到达的分段的序列号是初始序列号。 您需要跟踪它，为了保持32位 wrap seqnos之间转换。
- **将任何数据或流结束标记推送到StreamReassembler。**如果在TCPSegment的标头中设置了FIN标志，则表示有效负载的最后一个字节是整个流的最后一个字节。 请记住，StreamReassembler期望流索引从零开始。 您将必须unwrap seqnos才能产生这些。

##### ackno()

返回一个`optional<WrappingInt32>`，其中包含接收者尚不知道的第一个字节的序列号。 这是窗口的左边缘：接收方对将要接收的第一个字节有兴趣。 如果尚未设置ISN，请返回一个空的可选内容。

##### window_size()

返回“第一个未汇编的”索引（对应于ackno的索引）和“第一个不可接受的”索引之间的距离。

---

### volution of theTCPReceiverover the life of the connection

在TCP连接过程中，我们的TCPReceiver会经历一系列状态：从等待SYN（带有空ackno）到正在进行的流，再到完成的流，这意味着输入已在ByteStream上结束。 测试套件将检查您的TCPReceiver是否正确处理了传入的TCPSegments，并通过这些状态进行了扩展，如下所示。 （在实验4之前，您不必担心错误状态或RST标志。）

![](img-lab2\lab2-img-3.png)

---

### Development and debugging advice

1. 在文件tcpreceiver.cc中添加TCPReceiver的公共接口（以及您想要的任何私有方法或功能）。 您可以将任何你需要的的私有成员添加到intcpreceiver.hh中的TCPReceiver类里。
2. 编译后，您可以使用`make check_lab2`测试您的代码
3. 请努力使您的代码对要对其样式进行评级的CA可读。请对变量使用合理且清晰的命名约定。 使用注释来解释复杂或微妙的代码段。 使用“防御性编程”-明确检查函数或不变量的前提条件，如果遇到任何错误，则引发异常。 在设计中使用模块化-识别常见的抽象和行为，并在可能的情况下将它们排除在外。 重复的代码和巨大的功能模块将会使你的代码难以阅读。
4. 还请保持Lab 0文档中描述的“现代C ++”风格。 cppreference网站（https://en.cppreference.com）是一个很好的资源，尽管您不需要C ++的任何复杂功能即可进行这些实验。 （有时您可能需要使用move()函数来传递无法复制的对象。）
5. 如果您遇到段错误，那么确实出现了问题！ 我们希望您以一种安全的方式进行编程，来减少段错误的出现频率。（不要使用 malloc（），new，指针。并且在不确定的地方抛出异常的安全检查等）。 也就是说，要进行调试，您可以使用`cmake .. -DCMAKEBUILDTYPE = RelASan`配置构建目录，使编译器的“sanitizers”能够检测到内存错误和未定义的行为，并在发生错误时为您提供很好的诊断。。 您也可以使用**valgrind**工具。 您还可以配置`cmake .. -DCMAKEBUILDTYPE = Debug`并使用GNU调试器（gdb）。 但是请记住，这些选项（尤其是sanitizers”）会减慢编译和执行的速度！