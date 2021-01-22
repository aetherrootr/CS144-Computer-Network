## Lab0

---

### Set up GNU/Linux on your computer

解决方案：使用官网虚拟机镜像，并且挂载共享文件夹。可以实现在windows系统下编辑代码，在Linux系统下编译执行,感觉上跟wsl的体验差不多，不过性能比wsl低233333。未来可以考虑是否可以在wsl上复现相关的实验内容。

官方提供的配置文档基本上没有坑，可以放心食用(不过配置ssh的部分可能跟实际有所区别，需要自行研究)。参考文档：[https://stanford.edu/class/cs144/vm_howto/vm-howto-image.html](https://stanford.edu/class/cs144/vm_howto/vm-howto-image.html)

---

### Networking by hand

这个小节内要通过命令行完成两个任务。

1. 获取一个web页面
2. 发送一份电子邮件

这两个任务均需要自己通过手敲命令行解决。 并且它们都依赖于称为可靠双向有序字节流的网络抽象（其实就是TCP协议）。大致过程为在终端中键入一个字节序列， 并且最终将以相同的顺序将相同的字节序列传递给另一台计算机（服务器）上运行的程序。 服务器以自己的字节序列进行响应，并返回给终端。

#### Fetch a Web page

手动获取一个web页面。键入如下代码，并得到的反馈结果。

```
cs144@cs144vm:~$ telnet cs144.keithw.org http
Trying 104.196.238.229...
Connected to cs144.keithw.org.
Escape character is '^]'.
GET /hello HTTP/1.1                      #告诉服务器URL的路径部分
Host: cs144.keithw.org                   #告诉服务器URL的主机部分。(介于http://和第三个斜杠之间的部分。）
Connection:  close                       #告诉服务器完成请求后，理解关闭连接

HTTP/1.1 200 OK
Date: Wed, 20 Jan 2021 04:04:55 GMT
Server: Apache
Last-Modified: Thu, 13 Dec 2018 15:45:29 GMT
ETag: "e-57ce93446cb64"
Accept-Ranges: bytes
Content-Length: 14
Connection: close
Content-Type: text/plain

Hello, CS144!
Connection closed by foreign host.
```

随便捏一个学号玩一下。

```
cs144@cs144vm:~$ telnet cs144.keithw.org http
Trying 104.196.238.229...
Connected to cs144.keithw.org.
Escape character is '^]'.
GET /lab0/583924 HTTP/1.1
Host: cs144.keithw.org
Connection:  close

HTTP/1.1 200 OK
Date: Wed, 20 Jan 2021 04:12:42 GMT
Server: Apache
X-You-Said-Your-SunetID-Was: 583924
X-Your-Code-Is: 37575
Content-length: 110
Vary: Accept-Encoding
Connection: close
Content-Type: text/plain

Hello! You told us that your SUNet ID was "583924". Please see the HTTP headers (above) for your secret code.
Connection closed by foreign host.
```

#### Send yourself an email

由于我们无法使用斯坦福大学的邮箱，我们可以尝试使用QQ邮箱来做一个类似的实验。操作流程如下：

首先需要到QQ邮箱中开启stmp服务。

![](img-lab0\lab0-img-2.png)

由于QQ邮箱的安全限制，所以你需要获取stmp的专属访问密码。

![](img-lab0\lab0-img-1.png)

我们在用命令行登陆邮箱时，需要提供邮箱和密码的base64编码后的字符串。

可以到这个[网址](https://tool.oschina.net/encrypt?type=3)进行转换。

```
cs144@cs144vm:~$ telnet smtp.qq.com smtp               #连接QQ邮箱的smtp服务器
Trying 203.205.232.7...
Connected to smtp.qq.com.
Escape character is '^]'.
220 newxmesmtplogicsvrsza5.qq.com XMail Esmtp QQ Mail Server.
HELO mycomputer.qq.com                             #和smtp服务器打招呼
250-newxmesmtplogicsvrsza5.qq.com-9.22.14.83-59403186
250-SIZE 73400320
250 OK
auth login                              #登陆命令
334 VXNlcm5hbWU6                        
dGhpcyBpcyBhIGV4YW1wbGUgbWFpbA==        #用户名，一般是你的QQ邮箱（注意需要转换成base64编码）
334 UGFzc3dvcmQ6
dGhpcyBpcyBhIGV4YW1wbGUgcGFzc3dvcmQ=    #密码，需要在网页端QQ邮箱获取专门的访问密码（注意需要转换成base64编码）
235 Authentication successful
mail from:<XXX@qq.com>                 #输入发信人，就是你自己的QQ邮箱
250 OK.
rcpt to:<XXX@qq.com>                   #输入收信人
250 OK
DATA                                   #输入DATA告知服务器您已经准备好启动。
354 End data with <CR><LF>.<CR><LF>.
From:XXX@qq.com                          #发信人邮件地址
To:XXX@qq.com                            #收信人邮件地址
Subject: Hello from CS144 Lab 0!         #邮件标题
									  #以上三行为邮件首部，此处空一行以下表示邮件正文
this is a test email                      #邮件正文
.                                         #单独一行一个点，表示邮件结束
250 OK: queued as.                        #表示邮件已发出
QUIT                                      #断开连接
221 Bye.
Connection closed by foreign host.
```

之后你的QQ邮箱就会收到一封这样的邮件，表示实验成功了。

![](img-lab0\lab0-img-3.png)

####   Listening and connecting

建立一个简单的连接传递字符串。

Server

```
cs144@cs144vm:~$ netcat  -v -l -p 9090  #监听9090端口
Listening on [0.0.0.0] (family 0, port 9090)
Connection from localhost 47718 received!
test  			#2
quit             #4
^C               #5
cs144@cs144vm:~$
```

Client

```
cs144@cs144vm:~$ telnet localhost 9090
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
test          #1
quit          #3
Connection closed by foreign host.        #6
cs144@cs144vm:~$
```

---

### Writing a network program using an OS stream socket

在此热身实验的下一部分中，您将编写一个简短的程序，该程序通过Internet来获取Web页面。 您将利用Linux内核和大多数其他操作系统提供的功能：在两个程序之间*创建可靠的双向字节流的能力*，一个程序在您的计算机上运行，另一个程序在整个Internet（例如Web服务器）上 （例如Apache或nginx, **netcat**程序）。

此功能称为流套接字。 对于您的程序和Web服务器，套接字看起来像一个普通的文件描述符（类似于磁盘上的文件或stdin 或stdout这种I / O流）。 当连接两个流套接字时，写入一个套接字的任何字节最终都将以相同的顺序从另一台计算机上的另一个套接字中输出。

但是，实际上，Internet无法提供可靠的字节流服务。 取而代之的是，Internet唯一真正要做的就是尽“最大努力”将称为Internet数据报的短数据传递到目的地。 每个数据报都包含一些元数据（标头），这些元数据指定了诸如源地址和目标地址（它来自哪台计算机，以及它指向哪台计算机）之类的内容，以及一些要传递到目标计算机的有效负载数据（最多1500字节）。

 尽管网络尽力传递每个数据报，实际上数据报传输过程中可能会发送（1）丢失，（2）乱序发送，（3）传输过程中内容发送了更改，甚至（4）重复传送相同的内容。 将“尽力而为”数据报（Internet提供的抽象）转换为“可靠的字节流”（应用程序通常需要的抽象）通常是连接两端的操作系统的工作。

两台计算机必须配合使用 确保流中的每个字节最终都在适当的位置最终传递到另一侧的流套接字。 他们还必须告诉对方他们准备从另一台计算机接受多少数据，并确保发送的数据不超过另一边的接受容量（make sure not to send more than the other side is willing to accept）。 所有这些操作都是通过1981年制定的协议达成的，该协议称为传输控制协议（Transmission Control Protocol）或TCP。在本实验中，您将仅使用操作系统对TCP的预先支持。 您将编写一个名为“ webget”的程序，该程序将创建一个TCP流套接字，连接到Web服务器并获取一个页面，这与本实验前面的操作很相似。 在以后的lab中，您将通过自己实现传输控制协议以从不太可靠的数据报中创建可靠的字节流来实现此抽象的另一面。

#### Let’s get started—fetching and building the starter code

跟着给的流程执行就好，建议把项目放在一个共享文件夹中，这样可以在外部用自己喜欢的编辑器或IDE编辑。

这里有一个坑需要注意，git clone请在Linux端执行，在windows端执行有可能导致测试时出现问题！！！！

#### Modern C++: mostly safe but still fast and low-level

Lab作业将以当代的C ++风格完成，该风格使用最新（2011年）的功能来尽可能安全地进行编程。 这可能与您过去编写C ++的方式不同。 有关此样式的参考，请参见C ++ 核心指南（http://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines）。

这个风格的基本思想是确保每个对象都被设计为具有尽可能小的公共接口，进行大量的内部安全检查，并且不易使用不当，并且能够自动管理内存。 我们要避免“成对”的操作（例如malloc / free或new / delete），这可能某一个成对的操作的后半部分不会执行（例如函数一个函数可能会提前返回或抛出异常）。作为替代，当一个对象的创建发生在构造函数中，销毁发生在析构函数中。这种风格被称为“初始化资源获取（Resource acquisition is initialization）”或RAII。

特别地，我们希望您：

- 将位于https://en.cppreference.com上的语言文档作为参考资料。
- 切勿使用malloc()或free()。

- 切勿使用new或delete。

- 基本上永远不要使用原始指针（*），仅在必要时使用“智能”指针（unique_ptr或shared_ptr）。 （您无需在CS144中使用它们。）

- 避免使用模板，线程，锁和虚函数。 （您不需要在CS144中使用它们。）

- 避免使用C样式的字符串（char *str）或字符串函数（strlen()，strcpy()）。这些都是容易出错的。 请使用std :: string代替。

- 请勿使用C样式的强制类型转换（例如（FILE *）x）。 如果需要的话，请使用C ++ static_cast。（在CS144中通常不需要此方法。）

- 优先通过const来传递函数参数（例如const Address＆address）。

- 如果不需要对变量进行改变，就将变量设置为const。

- 如果不需要改变类的变量或参数，就将这些设置为const。

- 避免使用全局变量，并为每个变量提供尽可能小的范围。

- 在提交作页之前，请运行make format来规范编码样式。



使用Git：使用Git（版本控制）存储库来分发Lab -- 以此记录变更的方式，检查版本以帮助调试以及跟踪源代码的来源。**请在工作时进行频繁的小型提交，并使用提交消息标识更改的内容和原因。柏拉图式的理想是每次提交都应编译并朝着稳定的方向移动 越来越多的测试通过。** 进行小的“语义”提交有助于调试（调试每个提交都可以编译，并且消息描述了该提交确实很明确的事情，调试起来要容易得多），并且通过记录随着时间的推移稳步前进而保护您免受欺骗的侵害，这是一项有用的技能， 在包括软件开发在内的任何职业中提供帮助。 评分员将阅读您的提交信息，以了解您如何开发Lab解决方案。 如果您还没有学会如何使用Git，请在CS144办公时间寻求帮助或咨询教程（例如https://guides.github.com/introduction/git-handbook）。 最后，欢迎您将代码存储在GitHub，GitLab，Bitbucket等上的私有存储库中，但是请确保您的代码不可公开访问。

#### Reading the Sponge documentation

阅读相关的文档和代码没啥好说的。

#### Writing webget

coding，实现一个socket通信，按照流程测试即可。

```C++
void get_URL(const string &host, const string &path) {
    // Your code here.

    // You will need to connect to the "http" service on
    // the computer whose name is in the "host" string,
    // then request the URL path given in the "path" string.

    // Then you'll need to print out everything the server sends back,
    // (not just one call to read() -- everything) until you reach
    // the "eof" (end of file).

    // cerr << "Function called: get_URL(" << host << ", " << path << ").\n";
    // cerr << "Warning: get_URL() has not been implemented yet.\n";

    TCPSocket _Socket;

    _Socket.connect(Address(host, "http"));//建立连接
    _Socket.write("GET " + path + " HTTP/1.1\r\nHost: " + host + "\r\n\r\n");//传递命令

    _Socket.shutdown(SHUT_WR);//断开连接

    while (!_Socket.eof()) {//输出内容
        cout << _Socket.read();
    }

    _Socket.close();//关闭连接

    return;
}
```

---

### An in-memory reliable byte stream

到目前为止，您已经了解了*可靠字节流的抽象类*如何在Internet上进行通信时起到作用，即使Internet本身仅提供“尽力而为”（不可靠）数据报的服务。

本周的最后一个Lab ，您将在单个计算机的内存中给出一个抽象对象的实现。 （您可能在CS 110中做了类似的操作。）字节被写入“输入”端，并且可以以相同的顺序从“输出”端读取。 字节流是有限的：编写器可以结束输入，然后不能再写入任何字节。 当读取器读取到流的末尾时，它将到达“ EOF”（文件末尾），无法再读取任何字节。

您的字节流也将受到*流量控制*以限制其在任何给定时间的内存消耗。 该对象使用特定的“容量”进行初始化：在任何给定点它愿意存储在其自己的内存中的最大字节数。 字节流将限制写入器在任何给定时刻可以写入的数量，以确保该流不超过其存储容量。 当读取器读取字节并将其从流中排出时，允许写入器写入更多字节。 您的字节流供单线程使用-您不必担心因为并发导致的读者/写者问题，锁定或竞争条件。

需要明确的是：字节流是有限的，但是在编写者结束输入并完成流之前，字节流几乎可以任意长。 您的实现必须能够处理比容量更长的流。 容量限制给定点在内存中保留（写入但尚未读取）的字节数，但不限制流的长度。 一个容量只有一个字节的对象仍然可以承载长达数兆字节和数兆字节的流，只要写入器一次保持写入一个字节，并且读取器在允许写入器写入下一个字节之前读取每个字节即可。

 写者端接口：

```C++
// Write a string of bytes into the stream. Write as many
// as will fit, and return the number of bytes written.
size_t write(const std::string &data);

// Returns the number of additional bytes that the stream has space for
size_t remaining_capacity() const;

// Signal that the byte stream has reached its ending
void end_input();

// Indicate that the stream suffered an error
void set_error();
```

读者端接口：

```c++
// Peek at next "len" bytes of the stream
std::string peek_output(const size_t len) const;

// Remove ``len'' bytes from the buffer
void pop_output(const size_t len);

// Read (i.e., copy and then pop) the next "len" bytes of the stream
std::string read(const size_t len);

bool input_ended() const;	 // `true` if the stream input has ended
bool eof() const;			// `true` if the output has reached the ending
bool error() const;			// `true` if the stream has suffered an error
size_t buffer_size() const; // the maximum amount that can currently be peeked/read
bool buffer_empty() const; // `true` if the buffer is empty

size_t bytes_written() const;	// Total number of bytes written
size_t bytes_read() const;	// Total number of bytes popped
```

请打开libsponge / bytestream.hh 和 libsponge / bytestream.cc文件，并实现一个提供此接口的对象。 在开发字节流实现时，可以使用`make check_lab0`运行自动化测试。

接下来我们要做什么？在接下来的四个星期中，您将实现一个系统来提供相同的接口，不再存在于内存中，而是通过不可靠的网络。 这是传输控制协议（Transmission Control Protocol）。