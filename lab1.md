## Lab1

---

### Overview

在Lab0中，我们使用了Internet流套接字通过Linux的传输控制协议（TCP）的内置实现从网站获取信息并发送电子邮件。 即使基础网络仅提供“尽力而为”数据传输功能，该TCP仍设法实现了产生一对可靠的有序字节流（一个从您到服务器，一个在相反方向）。 我们的指的不可靠意思是：可能丢失，重新排序，更改或复制的数据短包。 您还自己在一台计算机的内存中实现了字节流抽象。 在接下来的四个星期中，您将实现一个TCP协议，以在由不可靠的数据报网络分隔的两台计算机之间提供字节流抽象。

> *我为什么要这么做？*
>
> 在一个无差异的可靠服务之上提供一个服务或一个抽象说明了网络中许多有趣的问题。在过去的40年里，研究人员和实践者们已经发现了如何在全球范围内传递各种各样的信息——信息、电子邮件、超链接文档、搜索引擎、声音和视频、虚拟世界、协作文件共享、数字货币互联网.TCP自己的角色，使用不可靠的数据报提供一对可靠的字节流，这是一个典型的例子。有一种合理的观点认为，TCP实现是世界上使用最广泛的非平凡计算机程序。

![](img-lab1\lab1-img-1.png)

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

在本实验中，您将编写负责该重组的数据结构：StreamReassembler。 它将受到若干个子字符串，字符串是的第一个字节代表该子字符的索引后面较大的部分是该子字符串的字节内容，它们一起包含在一个较大的流中。流中的每个字节都有自己的唯一索引，从零开始向上计数。 Stream重组器将拥有一个字节流作为输出：一旦重组器知道流的下一个字节，它将把它写入到ByteStream中。 所有者可以在需要时随时访问和读取ByteStream。

接口如下所示：

```c++
// 构造一个最多可存储“capacity”字节的“ StreamReassembler”。
StreamReassembler(const size_t capacity);

// 接收一个子字符串并将任何新的连续字节写入流中，
// 同时保持在“capacity”的内存限制之内。 字节数
// 超出容量则被丢弃。
//
// `data`: 子字符串
// `index` 指示“ data”中第一个字节的索引（按顺序放置）
// `eof`: 该子字符串的最后一个字节将是整个流中的最后一个字节
void push_substring(const string &data, const uint64_t index, const bool eof);

// 访问重新组装的ByteStream（Lab0中的代码）
ByteStream &stream_out();

// 已存储但尚未重新组合的子字符串中的字节数
size_t unassembled_bytes() const;

// 内部状态是否为空（输出流除外）？
bool empty() const;
```

> *我们为什么要这样做？*
>
> TCP对重排序和重复的鲁棒性来自它将字节流的任意摘录缝合回原始流的能力。在一个离散的可测试模块中实现这一点将使处理输入段变得更容易

重组器的完整（公共）接口由streamreassembler.hh头中的StreamReassembler类描述。 您的任务是实现此类。 您可以将所需的任何私有成员和成员函数添加到StreamReassembler类，但不能更改其公共接口。

### What’s the “capacity”?

您的push_substring方法将忽略字符串的任何部分，这将导致StreamReassembler超出其“容量”：内存使用的限制，即允许存储的最大字节数。 无论TCP发送方决定做什么，这都可以防止重组器使用无限制的内存量。 我们在下面的图片中对此进行了说明。 “容量”是两个方面的上限：

1. 已经重新组合的ByteStream中的字节数（下面以绿色显示），以及

2. “未装配的”子字符串可以使用的最大字节数（以红色显示）

![](img-lab1\lab1-img-2.png)

当您通过测试实施StreamReassemblerand工作时，您可能会发现这张图片很有用-“正确”行为并不总是很自然。

---

### FAQs

- 整个流中第一个字节的索引是什么？零。
- 我的实现应该有多有高的效率？请不要将此视为构建空间或时间效率低的数据结构的挑战——该数据结构将成为您TCP实现的基础。 可以预期的是，每个新的Lab 1测试都可以在不到半秒的时间内完成。
- 不一致的子字符串应如何处理？您可能会认为它们不是文本。 也就是说，您可以假设存在唯一的基础字节流，并且所有子串都是其（准确的）切片。
- 应何时将字节写入流？ 唯一不应该在流中存在字节的情况是，当在它之前还有一个字节尚未被“推送”时
- 提供给push_substring()函数的子字符串可以重叠吗？可以
- 我需要向StreamReassembler添加私有成员吗？ 子字符串可以以任何顺序到达，因此您的数据结构将必须“记住”子字符串，直到准备好将其放入流中为止，也就是直到之前的所有索引都被写入为止。
- 我们的重组数据结构可以存储重叠的子字符串吗？ 可以实现一个“接口正确的”重组器来存储重叠的子字符串。 但是，允许重组器执行此操作会破坏“容量”这一作为内存限制的概念。 我们将在分级时将重叠的子字符串的存储视为违反样式。

### Development and debugging advice

1. 您可以使用`make check_lab1`测试代码（编译后）。
2. 请重新阅读Lab 0文档中有关“使用Git”的部分，并记住将代码保存在主分支上分发的Git存储库中。 使用良好的提交消息来确定较小的提交，这些消息会标识更改的内容和原因。
3. 请努力使您的代码对要对其样式进行评级的CA可读。请对变量使用合理且清晰的命名约定。 使用注释来解释复杂或微妙的代码段。 使用“防御性编程”-明确检查函数或不变式的前提条件，如果有任何错误，请抛出异常。 在设计中使用模块化-识别常见的抽象和行为，并在可能的情况下将它们排除在外。 重复的代码块和巨大的功能将使您难以遵循代码。
4. 还请保持Lab 0文档中描述的“现代C ++”风格。 cppreference网站（https://en.cppreference.com）是一个很好的资源，尽管您不需要C ++的任何复杂功能即可进行这些实验。 （有时您可能需要使用move()函数来传递无法复制的对象。）
5. 如果您遇到段错误，那么确实出现了问题！ 我们希望您以一种安全的方式进行编程，来减少段错误的出现频率。（不要使用 malloc（），new，指针。并且在不确定的地方抛出异常的安全检查等）。 也就是说，要进行调试，您可以使用`cmake .. -DCMAKEBUILDTYPE = RelASan`配置构建目录，使编译器的“sanitizers”能够检测到内存错误和未定义的行为，并在发生错误时为您提供很好的诊断。。 您也可以使用**valgrind**工具。 您还可以配置`cmake .. -DCMAKEBUILDTYPE = Debug`并使用GNU调试器（gdb）。 但是请记住，这些选项（尤其是sanitizers”）会减慢编译和执行的速度！
6. 您可以使用`make clean`和`cmake .. -DCMAKEBUILDTYPE = Release`来重新构建系统。或者，如果您的构建确实卡住并且不确定如何修复它们，则可以擦除您的`build`目录（`rm -rf build`——请注意不要输入错误，因为这有可能导致灾难性的后果），然后请新建目录，然后再次使用`cmake ..`。

---

### 解决思路

这个lab需要我们实现一个流重组器。数据在网络中传播会被分割成若干个不同大小的数据包，由于网络的不可靠性，数据包到达的时间是不可控的，我们需要实现流重组器模块把这些数据包恢复成原来的顺序（这里不考虑丢包带来的影响）。

我们将受到一堆字节片段，这些片段将包含该数据包在数据中开始的位置，该数据包包含的数据是否为数据的最后一段，以及该数据包所包含的信息。我们可以将这个问题抽象为给定若干个字符串片段和它们在字符串中开始的位置，输出一个完整有序的字符串。我们还将面临字符串乱序，重复和重叠的情况，已经一些其它的极端情况，例如超出缓存区的大小，我们需要如何处理。

首先我们显然可以认为重复是一种特殊的重叠情况，从而减少我们讨论的成本，降低代码的复杂程度。然后我们也可以很容易想到每次我们需要重排字符串片段，只需要选取当前首位置最小的字符串片段即可，因此我们很容易想到使用小根堆来完成这个功能，我们遵循文档中的要求使用”现代C++“的要求可以使用`std::priority_queue<>`来达到堆的效果。我们只需要每次从堆顶取出一个字符串片段，判断这个片段的开头位置与当前已经排好序的位置关系，即可知道当前是否可以加入缓冲区(这部分功能在Lab0已经实现)。

stream_reassembler.hh

```c++
#ifndef SPONGE_LIBSPONGE_STREAM_REASSEMBLER_HH
#define SPONGE_LIBSPONGE_STREAM_REASSEMBLER_HH

#include "byte_stream.hh"

#include <cstdint>
#include <queue>
#include <string>

//! \brief A class that assembles a series of excerpts from a byte stream (possibly out of order,
//! possibly overlapping) into an in-order byte stream.
class StreamReassembler {
  private:
    // Your code here -- add private members as necessary.

    struct DataBlock {
        std::string data;
        size_t index;
        bool eof;

        bool operator<(const DataBlock &rhs) const { return index > rhs.index; }
    };

    ByteStream _output;  //!< The reassembled in-order byte stream
    size_t _capacity;    //!< The maximum number of bytes
    size_t _ok_pos;
    size_t _max_pos;
    bool _eof;

    std::priority_queue<DataBlock> BlockHeap;

  public:
    //! \brief Construct a `StreamReassembler` that will store up to `capacity` bytes.
    //! \note This capacity limits both the bytes that have been reassembled,
    //! and those that have not yet been reassembled.
    StreamReassembler(const size_t capacity);

    //! \brief Receive a substring and write any newly contiguous bytes into the stream.
    //!
    //! The StreamReassembler will stay within the memory limits of the `capacity`.
    //! Bytes that would exceed the capacity are silently discarded.
    //!
    //! \param data the substring
    //! \param index indicates the index (place in sequence) of the first byte in `data`
    //! \param eof the last byte of `data` will be the last byte in the entire stream
    void push_substring(const std::string &data, const uint64_t index, const bool eof);

    //! \name Access the reassembled byte stream
    //!@{
    const ByteStream &stream_out() const { return _output; }
    ByteStream &stream_out() { return _output; }
    //!@}

    //! The number of bytes in the substrings stored but not yet reassembled
    //!
    //! \note If the byte at a particular index has been pushed more than once, it
    //! should only be counted once for the purpose of this function.
    size_t unassembled_bytes() const;

    //! \brief Is the internal state empty (other than the output stream)?
    //! \returns `true` if no substrings are waiting to be assembled
    bool empty() const;
};

#endif  // SPONGE_LIBSPONGE_STREAM_REASSEMBLER_HH
```

stream_reassembler.cc

```c++
#include "stream_reassembler.hh"

// Dummy implementation of a stream reassembler.

// For Lab 1, please replace with a real implementation that passes the
// automated checks run by `make check_lab1`.

// You will need to add private members to the class declaration in `stream_reassembler.hh`

template <typename... Targs>
void DUMMY_CODE(Targs &&... /* unused */) {}

using namespace std;

StreamReassembler::StreamReassembler(const size_t capacity)
    : _output(capacity), _capacity(capacity), _ok_pos(0), _max_pos(0), _eof(false), BlockHeap() {}

//! \details This function accepts a substring (aka a segment) of bytes,
//! possibly out-of-order, from the logical stream, and assembles any newly
//! contiguous substrings and writes them into the output stream in order.
void StreamReassembler::push_substring(const string &data, const size_t index, const bool eof) {
    if (_eof)
        return;

    _max_pos = max(_max_pos, index + data.length());

    DataBlock _data_block = DataBlock{data, index, eof};
    BlockHeap.push(_data_block);

    while (!BlockHeap.empty() && !_eof) {
        DataBlock _block = BlockHeap.top();

        if (_block.index <= _ok_pos) {
            int _ignore_length = _block.data.length() + _block.index - _ok_pos;

            if (_output.remaining_capacity() && _ignore_length > 0) {
                _ok_pos -= _output.buffer_size();
                _output.write(_block.data.substr(_block.data.length() - _ignore_length));
                _ok_pos += _output.buffer_size();
            }
        } else {
            break;
        }

        BlockHeap.pop();
        if (_block.eof && _ok_pos - _block.index + _block.data.length() <= _capacity) {
            _eof = true;
            _output.end_input();
        }
    }
}

size_t StreamReassembler::unassembled_bytes() const { return _ok_pos == 0 ? (_max_pos - 1) : (_max_pos - _ok_pos); }

bool StreamReassembler::empty() const { return _ok_pos == 0; }
```

#### 一些边界条件

1. 当读到的数据包的eof标志为true时，不代表一定eof了，可能在转载进入缓冲区时丢弃部分信息导致eof没有进入缓冲区（详细可看stream_reassembler.cc的46行）。
2. 同样把字符串装载如缓冲区时要考虑丢弃部分信息的情况，frist_unassembled的偏移不一定是你插入字符串片段的长度（详细可看stream_reassembler.cc的37和39行）。



我们接着可以来讲讲这个图：

![](img-lab1\lab1-img-2.png)

从Lab1来理解这张图我们可以把绿色的部分理解为我们在Lab0实现的缓冲区，红色的部分理解为我们在Lab1实现的流重组器中的那个小根堆。这样看其实很多实现方面的细节其实我们都可以想得到。我个人觉得Lab0提供的接口还不太完备，Lab0还能通过更加完备的接口，来完全封装这个流缓冲区。