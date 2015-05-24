---
layout: post
title:  "读《UNIX网络编程 卷1：套接字联网API》[上]"
date:   2014-10-09 10:56:33
categories: tech
keywords: "UNIX网络编程, unix network programming, UNP, tcp, udp, 三次握手, 四次挥手"
description: "本文介绍了TCP和UDP的工作过程，TCP连接建立时的三次握手、连接断开时的四次挥手，非正常网络状态下TCP的工作状况"
---

最近看了《UNIX网络编程 卷1：套接字联网API》，
英文名叫Unix Network Programming啦，后来上网查了查，
一般都叫[UNP](http://en.wikipedia.org/wiki/UNIX_Network_Programming)逼格会高一点，
就像[APUE](http://en.wikipedia.org/wiki/Advanced_Programming_in_the_Unix_Environment)一样。
他们的作者都是[W. Richard Stevens](http://en.wikipedia.org/wiki/W._Richard_Stevens)。
另外，他也是[TCP/IP Illustrated](http://en.wikipedia.org/wiki/TCP/IP_Illustrated)的作者。
靠，看完作者简介，简直崇拜得五体投地了。

说说这本书中比较让我印象深刻的内容吧，我只看了书中关于关于TCP和UDP的主要部分，略过了一些章节，所以可能有一些遗漏。

+ TCP和UDP的工作过程
+ TCP连接在“非正常”情况下的工作状况
+ 各种I/O模型（阻塞/非阻塞/IO复用/信号驱动/异步）
+ 守护进程和inetd的工作原理
+ 服务器程序设计范式
+ 客户端程序设计范式

#### **TCP和UDP的工作过程** #####

UDP的工作过程是简单的，仅仅将用户数据封装到一个IP数据报中发送到目的地而已，而不关注其他方面。

TCP却是一个极其复杂的协议，以下只是冰山一角

+ ##### **建立连接的三次握手** #####

  1. 主动方发送**`（SYN J）`**，进入**`SYN_SENT`**状态
  2. 被动方收到**`（SYN J）`**，并往回发送**`（SYN K, ACK J+1）`**，进入**`SYN_RCVD`**状态
  3. 主动方收到**`（SYN K, ACK J+1）`**，并往回发送**`（ACK K+1）`**，进入**`ESTABLISHED`**状态
  4. 被动方收到**`（ACK K+1）`**，也进入**`ESTABLISHED`**状态
  
  以上过程如下图所示：

  ![establish](/image/establish.png)

  注意到在TCP三次握手的过程中，服务器有这么一条：

  > 2\. 被动方收到**`（SYN J）`**，并往回发送**`（SYN K, ACK J+1）`**，进入**`SYN_RCVD`**状态

  服务器进入**`SYN_RCVD`**状态（此时连接称为半开连接）后，应当期待再收到一个ACK。
  如果超时未收到客户端的ACK，服务器将重发**`（SYN K, ACK J+1）`**。
  于是，就有一种叫做**[`SYN Flooding`](http://en.wikipedia.org/wiki/SYN_flood)**的攻击方式。
  攻击者向服务器高速发送**`（SYN J）`**（而且可以将SYN分节中的IP地址设为随机数），
  并且在随后收到服务器回复的**`（SYN K, ACK J+1）`**之后不再继续回复，
  这使得服务器上存在很多的半开连接，这些半开连接一般情况下会持续63秒
  （[在Linux下，默认重试次数为5次，重试的间隔时间从1s开始每次都翻倍，5次的重试时间间隔为1s, 2s, 4s, 8s, 16s，第5次发出后还要等32s都知道第5次也超时了，所以，总共需要 1s + 2s + 4s+ 8s+ 16s + 32s = 63s，TCP才会把断开这个连接](http://coolshell.cn/articles/11564.html)）。
  它的危害有两方面，一方面自然是占用了服务器的资源；另一方面是填充了半开连接的队列，使得合法的SYN分节无法排队。

  根据SYN Flooding的攻击原理，它的防范主要有以下措施：

  1. 过滤掉最大嫌疑攻击的IP或IP段
  2. 将**`tcp_synack_retries`**设为0，表示回应第二个握手包**`（SYN K, ACK J+1）`**给客户端后，如果收不到ACK，不进行重试，加快回收“半开连接”。
  3. 将**`tcp_max_syn_backlog`**参数根据内存情况适当调大，该参数一般指的是维护的半开连接的队列的长度（不同OS不一样）。
  4. 设置**`tcp_abort_on_overflow`**选项，处理不过来就直接拒绝掉。
  
+ ##### **断开连接的四次握手** #####

  1. 主动方发送**`（FIN M）`**，进入**`FIN_WAIT_1`**状态
  2. 被动方收到**`（FIN M）`**，并往回发送**`（ACK M+1）`**，进入**`CLOSE_WAIT`**状态
  3. 主动方收到**`（ACK M+1）`**，进入**`FIN_WAIT_2`**状态
  4. 被动方发送**`（FIN N）`**，进入**`LAST_ACK`**状态
  5. 主动方收到**`（FIN N）`**，并往回发送**`（ACK N+1）`**，进入**`TIME_WAIT`**状态
  6. 被动方收到**`（ACK N+1）`**，进入**`CLOSED`**状态
  7. 主动方在**`TIME_WAIT`**状态中超时后，进入**`CLOSED`**状态

  
  以上过程如下图所示：

  ![close](/image/close.png)

  其实就是2次，只不过TCP是全双工的，所以，发送方和接收方都需要FIN和ACK。
  只不过，有一方是被动的，所以看上去就成了所谓的4次挥握手。

  注意到最后有这么一条涉及到**`TIME_WAIT`**的状态

  > 7\. 主动方在**`TIME_WAIT`**状态中超时后，进入**`CLOSED`**状态
  
  需要经过一个**`TIME_WAIT`**超时的状态而不是直接进入**`CLOSED`**的原因有两个，一是确保有足够的时间让对端收到ACK，二是允许老的分节在网络中慢慢的消逝。

  然而，如果系统中存在着大量的短链接，那么大量的**`TIME_WAIT`**状态就会成为系统的累赘。网上一些资料提到的**`tcp_tw_reuse`**和**`tcp_tw_recycle`**选项来解决这个问题，但是最好还是别乱用，好像[coolshell](http://coolshell.cn/articles/11564.html)中有提到过，可能会出很多诡异的问题。还可以调整**`tcp_max_tw_buckets`**，当并发的**`TIME_WAIT`**过多时，会直接把多的给destory掉，然后在日志里打一个警告。引用一句“[其实，TIME_WAIT表示的是你主动断连接，所以，这就是所谓的no zuo， no die](http://coolshell.cn/articles/11564.html)”。

#### **TCP连接在“非正常”情况下的工作状况** #####

+ ##### **服务器进程终止** #####

  首先，服务器进程终止（收到**`SIGKILL`**信号）。作为进程中止处理的工作之一，该进程所有打开着的描述符将被关闭，这会导致向对端（客户端）发送**`（FIN N）`**，而客户端则回复**`（ACK N+1）`**，这就是TCP断开连接的前半部分。

  然后，此时客户端收到**`（FIN N）`**并不意味着连接断开（虽然在这个例子中，确实断开了），只是意味着服务器不再向客户端发送数据了，客户端还可以继续向服务器发送数据。如果此时客户端还继续向服务器发送数据，服务器TCP将发现之前的打开该套接字的进程已终止，于是回应一个**`RST`**。客户端在收到这个**`RST`**之前的read操作将会返回EOF，在收到这个**`RST`**后的read操作会返回**`ECONNRESET`**错误，在收到这个**`RST`**后的write操作会使当前进程收到**`SIGPIPE`**信号。

  以上过程如下图所示：

  ![server_kill](/image/server_kill.png)

+ ##### **服务器主机崩溃** #####

  服务器主机崩溃的意思是，没有任何预兆，来不及在网络上发送任何消息，主机就无法工作了。这种情况等价于直接切断网络，或者通俗的说，可以直接拔掉网线来模拟这一情况。

  这时，如果客户端向服务器发送数据，后调用read操作，TCP会一直等待服务器的ACK确认消息，并且不断的超时重传（按照Berkeley的实现，重传12次，共需9分钟），直到到达重传次数，返回**`ETIMEOUT`**错误。如果是由中间的路由器判定服务器主机不可达，响应“destination unreasonable”的ICMP消息，将返回**`EHOSTUNREACH`**和**`ENETUNREACH`**错误。

+ ##### **服务器主机崩溃后重启** #####
  
  重启之后的服务器已经丢失了之前的TCP信息，所以即使收到了客户端发来的TCP数据，也会回复**`RST`**，往后的情况和“服务器主机崩溃”中提到的类似。

+ ##### **服务器主机关机** #####
  
  Unix系统关机时，init进程通常会给其他进程发送**`SIGTERM`**信号，然后等待10s左右给仍在运行的进程发送**`SIGKILL`**信号。所以如果进程不捕获**`SIGTERM`**信号，则将由**`SIGKILL`**信号终止，和“服务器进程终止”中提到的类似。

再往后的I/O模型、守护进程和inetd的工作原理、服务器程序设计范式、客户端程序设计范式过几天有时间再写吧。
