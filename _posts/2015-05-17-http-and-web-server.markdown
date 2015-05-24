---
layout: post
title:  "Web Server 和 HTTP协议"
date:   2015-05-17 13:58:32
categories: jekyll update
---

一直在找实习，有点什么东西直接就在evernote里面记了，也没时间来更新到这里。找实习真是个蛋疼的事，一直找的是困难模式的C++的后台开发这种职位，主要是因为其他的更不会了。虽然找的是C++的职位，但是我的简历有俩项目都是php的，因为老赵的项目就是用php做网站。最近越来越感觉这样的简历不靠谱，想换个C++的和网络有关的多线程的项目吧。所以最近准备点几个网络和多线程的技能点。于是我看了[tinyhttpd](http://sourceforge.net/projects/tinyhttpd/)、[LightCgiServer](https://github.com/imyouxia/LightCgiServer)和吴导的[husky](https://github.com/yanyiwu/husky)。基本上对着吴导的husky抄了个[paekdusan](https://github.com/aholic/paekdusan)，但是也不能纯粹的抄一遍啊，所以还是改了一些小东西，大的框架没变。主要的改变包括以下几方面：

+ 在线程池部分中，使用C++11的thread替代了pthread，从而实现跨平台的目标
+ 在支持并发的队列中，使用C++11的mutex和lock替代了pthread的mutex和lock，从而实现跨平台的目标
+ 在socket部分，使用了预编译宏的方式，从而实现跨平台的目标
+ 接收数据部分更健壮，以面对不能一次性读完一个HTTP头部的情况；发送也一样
+ 实现了一个具有简易的KeepAlive策略的HTTP服务器
+ 实现了一个静态文件的HTTP服务器

#### **tinyhttpd和LightCgiServer** #####

首先，还是先介绍一下tinyhttpd吧。网上的评价还是很高的，能让人仅从500-600行的代码中了解HTTP Server的本质。
贴一张tinyhttpd的流程图吧：

![tinyhttpd](/image/tinyhttpd.jpg)

关于tinyhttpd更详细的信息，大家还是直接去看代码吧，因为真的很易读、易懂。tinyhttpd的代码给人的感觉就是，怎么易读、易懂怎么来，例如服务器回复一个`501 Method Not Implemented`的response是这么写的，看到我就惊呆了，只能怪我以前看过的代码太少，我第一反应就是先sprintf到一个长的buff里面，然后一起send，但是它这样的写法确实更加易懂、易读。
  
    void unimplemented(int client) {
        char buf[1024];
  
        sprintf(buf, "HTTP/1.0 501 Method Not Implemented\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, SERVER_STRING);
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "Content-Type: text/html\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "<HTML><HEAD><TITLE>Method Not Implemented\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "</TITLE></HEAD>\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "<BODY><P>HTTP request method not supported.\r\n");
        send(client, buf, strlen(buf), 0);
        sprintf(buf, "</BODY></HTML>\r\n");
        send(client, buf, strlen(buf), 0);
    }


除此之外，值得一提的是tinyhttpd实现的是一个CGI Server的功能，但是在CGI的功能上实现得比较简陋，[LightCgiServer](https://github.com/imyouxia/LightCgiServer)实现得更完整一些，关于CGI Server更详细的情况请看[CGI Server](http://armsword.com/2014/05/18/light-cgi-server/)

#### **husky和paekdusan** #####

正如本文开头所说，大的程序结构上，paekdusan基本是对着husky抄的，只是做了一些小的改变。程序在大的结构上，可以看作是一个**`生产者消费者模型`**。

先看一个不伦不类的流程图：

![paekdusan-flow-chart](/image/paekdusan-flow-chart.png)

从上图可以看出，主线程是**`生产者`**，线程池中的线程们是**`消费者`**，它们之间通过task队列来通信。主线程作为**`生产者`**，accept成功返回之后，将处理该client的task添加到task队列中，然后继续accept等待client的到来；线程池的线程们作为**`消费者`**，不断的从task队列中取出task，调用task的run接口。

值得注意的是，task队列是一个**`BoundedBlockingQueue`**，也就是说，task队列是一个有容量限制，并且阻塞的队列。当消费者试图从task队列中取task时，如果task队列是空的，则消费者会被阻塞，直到生产者往task队列中放入task，将消费者唤醒。同样的，当生产者试图向task队列中放入task时，如果task队列是满的，则生产者会被阻塞，直到消费者从task队列中取出task，将生产者唤醒。

再看一个不伦不类的时序图：

![paekdusan-sequence-diagram](/image/paekdusan-sequence-diagram.png)

这里要求task实现了run接口，task队列的设计可以认为是**`command模式`**的实践。看上去有很多类，但是其实是因为每个类的功能比较单一，程序只是把一些功能单一的类组合在一起了而已，其实类之间的耦合性比较低。

具体实现的代码见这里[paekdusan](https://github.com/aholic/paekdusan)

#### **问题记录** #####

+ ##### **HTTP协议的基本格式** #####
  
  HTTP的request的第一部分是request line，以空格分割得到的三部分依次是method，URI和version

  HTTP的request的第二部分是header，header以\r\n结尾，header中的每一行也以\r\n结尾，也就是说，当header是空时，以一个\r\n结尾；当header不空时，一定是以两个连续的\r\n结尾的。heder中的每一行格式是 key : value，其中value可以是空，所以简单的说，header是一个map，键和值之间用:分隔，键值对之间用\r\n分隔，在map的最后还有一个\r\n。值得注意的是，cookie是在header里面的。

  HTTP的request的第三部分是body，协议规定body后面不能再有其他字符，所以body不能靠去找\r\n来结束，要靠header里面的content-length来指明，content-length就是body的字节数。

  HTTP的response的第一部分是response line，以空格分割得到的三部分依次是version，status code和Reason Phrase

  HTTP的response的第二部分是header，格式和request类似

  HTTP的response的第三部分是body，格式和request类似

  另外，还有一点，我不知道request里面有没有可能出现，反正在response里面是会出现的，那就是如果header中指明了transfer-coding是chunked，那么body将会是一串chunked的块。在HTTP协议的rfc2616中是这么定义chunk-body的格式的：

      Chunked-Body   = *chunk
                       last-chunk
                       trailer
                       CRLF

      chunk          = chunk-size [ chunk-extension ] CRLF
                        chunk-data CRLF
      chunk-size     = 1*HEX
      last-chunk     = 1*("0") [ chunk-extension ] CRLF

      chunk-extension= *( ";" chunk-ext-name [ "=" chunk-ext-val ] )
      chunk-ext-name = token
      chunk-ext-val  = token | quoted-string
      chunk-data     = chunk-size(OCTET)
      trailer        = *(entity-header CRLF)
      
  也就是说chunk由四部分组成，首先是若干个chunk块（每个chunk块由chunk-size，可选的chunk-extension，\r\n, chunk-data 和 \r\n组成），接着是last-chunk块（chunk-size是0，没有chunk-data的特殊chunk块），然后是trailer(若干和header一样格式的数据组成)，最后是一个\r\n。其实不看中间“可选的chunk-extension”还是比较简单的。

+ ##### **KeepAlive的实现** #####

  在之前husky的代码中，server端发送了response之后，就close socket了，即关闭该socket，如果client需要再次发送http request需要再次建立一个新的tcp连接。而打开一个常见的网页，通常有很多http request从client发送到server，那么就需要很多次tcp的建立和断开，比较低效。之所以用KeepAlive就是为了避免多次请求需要重复的建立TCP连接，也就是说server端发送完response之后，不关闭连接，而是在该连接上继续等待数据。KeepAlive在HTTP1.1是默认开启的，如果要关闭，需要在header中声明connection: close。

  但是朴素的KeepAlive会引起一些问题，例如client一直不断开连接，那么和client的连接一直保持，client多了的时候，新来的client无法获得server的资源，所以需要一些其他的折衷。例如如果接下来的5s内都没有收到数据则断开连接，或者是接下来的5s内服务器接收了100个客户端请求就断开连接。

  由于“5s内服务器接收了100个客户端请求就断开连接”这需要在根据一个线程外部的信息控制线程的运行，使得线程运行过程中对于外部的以来过多，故而paekdusan没有这么实现，而是在同一个连接上接收了50个http request之后断开连接。另外paekdusan还实现了“一个连接的持续时间超过5s就断开连接”，具体来说是这样的，recv超时时间1s，每次recv完数据之后判断距离第一次recv的时间是否超过5s，超过则断开连接。详见如下代码：

      //简单起见 删除了一些处理不完整http请求的代码，并且简化了now 和 startTime的设置
      //详见https://github.com/aholic/paekdusan/blob/master/KeepAliveWorker.hpp
      while ((now - startTime <= 5) && requestCount > 0) {
          recvLen = recv(sockfd, recvBuff, RECV_BUFFER_SIZE, 0);
          if (recvLen <= 0 && getLastErrorNo() == ERROR_TIMEOUT) {
              LogInfo("recv returns %d", recvLen);
              continue;
          }
              
          if (recvLen <= 0) {
              LogError("recv failed: %d", getLastErrorNo());
              break;
          }

          //do with recvBuff, get a http request

          requestCount--;
      }
  
  但是由于使用的是阻塞的recv，所以实现的不是非常合理，存在一些问题。
  例如此时恰好距离第一次recv只有4.99秒，所以(now - startTime <= 5)满足，继续进入while循环，然后阻塞在recv上，recv设置的超时是1秒，那么其实最后跳出while循环的时间距离第一次recv已经过去了5.99秒。暂时没想到什么好办法，因为把recv设置成理解返回的话，while循环的次数太多，效率也不高。所以最好是要有一种通知的机制。

+ ##### **CGI Server** #####
  
  CGI Server一般是要fork一个进程来执行http request的URI中指定的CGI Script的，并且通过环境变量，向CGI Script传递本次请求的信息，具体怎么做可以看看[这篇文章](http://armsword.com/2014/05/18/light-cgi-server/)。但是注意到paekdusan是一个多线程的服务器，所以这里涉及到多线程和多进程，这是个很蛋疼的情况。多线程和多进程的混合会有很多问题，[这篇文章](http://www.linuxprogrammingblog.com/threads-and-fork-think-twice-before-using-them)有详细的介绍，好吧，你可能会发现它被墙了，那我还是简单的介绍一下会有什么问题吧。

  首先需要说明的是，在一个子线程中调用fork会发生什么：产生的子进程中只会有一个线程，也就是调用fork的这个线程。
  
  那么问题来了，假如父进程中的其他线程获取了一个锁，正在改线程间共享的数据，这时共享数据处于半修改状态。但是在子进程中，其他线程都消失了，那这些共享数据的修改怎么办？并且，锁的状态也得未定义了。另外，即使你的代码是线程安全的，你也不能保证你用的Lib的实现是线程安全的。

  所以唯一合理的在多线程的环境下使用多进程的情况，只有fork之后立即exec，也就是马上讲子进程替换成一个新的程序，这样的话，子进程中所有的数据都变得不重要，都抛弃了。所以，其实多线程的CGI Server也算是合理，但是需要注意安全性问题。因为打开的子进程默认是继承了父进程的文件描述符的，也就是说，子进程可以有父进程对文件的读写权限。

  我大概知道的就这么多，更详细的还是翻墙去读原文吧。

+ ##### **C++11的thread** #####

  C++11的thread用起来感觉比以前的linux上的pthread或者是windows上的beginthread都好用太多来，来一段简短的代码展示一下基本用法吧。

      void sayWord(const string& word) {
          for (int i = 0; i < 1000; i++) {
              cout << word << endl;
          }
      }
      void saySentence(const string& sentence) {
          for (int i = 0; i < 1000; i++) {
              cout << sentence << endl;
          }
      }

      int main() {
          string word = "hello";
          string sentence = "this is an example from cstdlib.com";
          thread t1(std::bind(sayWord, ref(word)));
          thread t2(saySentence, ref(sentence));

          t1.join();
          t2.join();
          return 0;
      }

  运行以上代码，会发现交替输出“hello”和“this is an example from cstdlib.com”。注意到t1和t2的构造参数看起来怪怪的，“std::bind(sayWord, ref(word))”和“saySentence, ref(sentence)”，主要线程函数的参数是引用，有个模版里面的bind在这，我也很难解释清楚，感觉模版叼叼的。另外，以非静态类成员函数创建线程时，需要在参数中带上this或者是ref一个对象的实例，不然无法调用。

+ ##### **C++11的mutex和lock** #####

  注意到上面thread的代码其实是有问题的，两个线程交替输出的东西可能会混合，所以要加锁。C++11的mutex和lock也很好用。

      mutex mtx;
      void sayWord(const string& word) {
          for (int i = 0; i < 1000; i++) {
              lock_guard<mutex> lock(mtx);
              cout << word << endl;
          }
      }
      void saySentence(const string& sentence) {
          for (int i = 0; i < 1000; i++) {
              lock_guard<mutex> lock(mtx);
              cout << sentence << endl;
          }
      }

  代码里面用的是lock_guard<mutex>，lock_guard就是在构造函数里面调用mutex的lock方法，析构函数里面调用mutex的unlock方法，用起来比较方便。unique_lock和lock_guard类似，但是多了一些其他的成员函数来配合其他的mutex类。关于mutex和lock的用法，[这篇博客](http://www.cnblogs.com/haippy/p/3237213.html)说的比较详细。主要和pthread里面的lock的区别就是，在pthread中，重复获取一个已经获得的lock不会报错，而在C++11中会报错。

+ ##### **C++11的condition_variable** #####

  在paekdusan的task队列是**`BoundedBlockingQueue`**，也就是说有阻塞和唤醒的操作。所以涉及到condition_variable。condition_variable主要用两个函数：**`wait(unique_lock<mutex>& lck)`**和**`notify_one()`**。线程被wait阻塞时，会先调用lck.unlock()函数释放锁；线程被notify_one唤醒时，会调用lck.lock()获取锁，以回复当时wait前的样子。关于condition_variable的用法，[这篇博客](http://www.cnblogs.com/haippy/p/3252041.html)说的比较详细

+ ##### **windows和linux上socket的API的不同点** #####

  1. windows上需包含winsock.h；linux上需包含cerrno，sys/socket.h，sys/types.h，arpa/inet.h和unistd.h
  2. windows上调用socket之前要调用WSAStartup，并且用#pragma comment(lib,"Ws2_32")链接Ws2_32.lib
  3. windows上关闭socket的函数叫做closesocket；linux上叫做close
  4. windows上获取错误码用GetLastError()；linux上查看全局变量errno，错误码的意义也不一样
  5. windows上设置SO_RCVTIMEO和SO_SNDTIMEO选项时，单位是毫秒；linux是秒
  6. windows上accept的原型是accept(SOCKET, struct sockaddr*, int*)；linux上accept的原型是accept(int, struct sockaddr*, socklen_t*)

  我遇到的就这些，估计还有很多，只是我还没遇到。

#### **相关阅读** ####
\[1\]：[Threads and fork(): think twice before mixing them](http://www.linuxprogrammingblog.com/threads-and-fork-think-twice-before-using-them)    
\[2\]：[RFC2616](http://www.ietf.org/rfc/rfc2616.txt)    
\[3\]：[写了一个简单的CGI Server](http://armsword.com/2014/05/18/light-cgi-server/)    
\[4\]：[tinyhttpd源码分析](http://blog.sina.com.cn/s/blog_a5191b5c0102v9yr.html)    
\[5\]：[HTTP协议头部与Keep-Alive模式详解](https://www.byvoid.com/blog/http-keep-alive-header/)    
\[6\]：[C++11并发指南](http://www.cnblogs.com/haippy/)    
