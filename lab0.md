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