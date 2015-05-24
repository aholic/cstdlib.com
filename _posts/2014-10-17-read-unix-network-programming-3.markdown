---
layout: post
title:  "读《UNIX网络编程 卷1：套接字联网API》[下]"
date:   2014-10-17 13:58:32
categories: tech
keywords: "服务端程序结构，客户端程序结构"
description: "本文介绍了常见的服务端程序结构和客户端程序结构，总结了各种结构的优缺点"
---

在[上一篇](/tech/2014/10/14/read-unix-network-programming-2/)中，
主要介绍各种I/O模型和守护进程以及inetd的工作原理。
在这一篇中，介绍介绍贯穿全书的客户端/服务器程序设计范式。

+ TCP和UDP的工作过程
+ TCP连接在“非正常”情况下的工作状况
+ 各种I/O模型（阻塞/非阻塞/IO复用/信号驱动/异步）
+ 守护进程和inetd的工作原理
+ 客户端程序设计范式
+ 服务器程序设计范式

#### **客户端程序设计范式** #####

+ ##### **停-等的迭代客户端** #####
  先看一段短短的代码，一个简单的回射（echo_back）客户端的核心代码如下：

      char sendLine[MAX_LEN], recvLine[MAX_LEN];
      while (fgets(sendLine, sendLine, stdin)) {
          write(sockFd, sendLine, strlen(sendLine));
          if (readline(sockFd, recvLine, MAX_LEN) == 0) {
            //error
          }
          fputs(recvLine, stdout);
      }

  这个客户端就是这样，停下来等用户输入，停下来等服务器回复，并且与用户交互和与服务器交互的过程是不能同时进行的，是顺序排列的。
  这样的客户端的优点就是编码简单，除了这个，实在是想不出其他优点了。
  缺点就很多了（不然其他的设计方法不就没啥意义了？）：
  在等待用户输入时，无法知晓对端关闭连接等网络事件，比如第5秒时，对端关闭了连接，但是到第10秒用户才输入，这时候readline才返回错误；而且停-等的模式使得它在批处理输入的情况下，效率极低。

+ ##### **I/O复用的另一种迭代客户端** #####

  “停-等的迭代客户端程序”中，无法实时的知晓网络状况的问题的核心在于
  面临着多个事件（网络和用户输入），却只阻塞于一个事件（用户输入）上。
  为了解决这个问题，回顾一下[I/O复用模型](/tech/2014/10/14/read-unix-network-programming-2/#io-2)，
  可以调用select/poll等函数，同时等待多个文件描述符（套接字描述符和用户输入描述符不就是两个文件描述符吗？）的状态变化，这种设计方法就可以同时检测多个文件描述符的状态。一种比较典型的实现方式如下：
  
      fd_set allset, rset;  //文件描述符集合，all，read
      int maxfdp1 = -1;  // = maxfd + 1
      FD_ZERO(&allset);  //清空allset
      FD_SET(fd0, &allset); maxfdp1 = fd0 > maxfdp1 ? fd0 : maxfdp1;
      ...//向allset中添加需要监视的文件描述符
      FD_SET(fd5, &allset); maxfdp1 = fd5 > maxfdp1 ? fd5 : maxfdp1;
      maxfdp1 += 1;

      while (1) {
          rset = allset;
          nReady = select(maxfdp1, &rset, NULL, NULL, NULL);
          if (FD_ISSET(fd0, &rset)) {
              // fd0上有数据可读，完成fd0相关的工作
          }
          if (FD_ISSET(fd1, &rset)) {
              // fd1上有数据可读，完成fd1相关的工作
          }
      }

  其实说白了，就是一个select函数的实践，如果你知道select函数咋用，这种设计方法就这样了。

+ ##### **非阻塞I/O的客户端** #####

  其实非阻塞的I/O客户端已经可以比较正确的工作了，不过还是存在着一些问题，
  完成某个文件描述符相关的工作假如比较耗时，
  那就会影响其他文件描述符上的工作。
  举个例子吧，用户输入字符串，而套接字的发送缓冲区已满（可能是网络较慢），
  那么向该套接字的写操作将阻塞，此时，套接字接收到数据，就没法读了（select所在的循环还没执行完1遍）。
  所以换非阻塞的方式来解决这个问题。使用非阻塞的方式时，需要在标准输入输出和套接字之间维护缓冲区。
  因为对于非阻塞的读（写）操作，并不会等指定的字节数全部从内核中拷贝出来（全部拷贝到内核）才返回，
  而立即返回成功的字节数，所以需要缓冲区来串联标准输入输出和套接字。
  想要看看具体缓冲区是怎么工作的，可以看看该书的16.2节。

  除了非阻塞的读写可以很好的发挥程序的动态优势，还有非阻塞的connect。
  回顾常见的阻塞connect，一个connect需要一个[RTT](http://en.wikipedia.org/wiki/Round-trip_delay_time)时间，而这个时间从几毫秒到几秒不等。
  在这段时间内，可以做很多事；而且对于要同时建立多个connect的客户端来说，
  使用非阻塞connect的等待多个RTT时间简直就是噩梦；另外还可以用select的超时来实现connect上的超时限制。

  而非阻塞的connect具有这样的特性：在一个非阻塞的TCP套接字上调用connect时，connect立即返回一个EINPROGRESS，
  但是已经发起的TCP三路握手继续进行。这样的话，接着使用select检测这个连接成功失败的条件就可以了。大致代码如下：
  
      n = connect(sockfd, ...);
      if (n < 0 && errno != EINPROGRESS) return -1;
      
      if (n == 0) goto done; //RTT时间很短，connect立即完成

      FD_ZERO(&rset);
      FD_SET(sockfd, &rset);
      wset = rset;
      tval.tv_sec = ...;
      tval.tv_usec = ...;
      
      n = select(sockfd+1, &rset, &wset, NULL, tval);
      if (n == 0) { // 超时了
          close(sockfd);
          errno = ETIMEOUT;
          return -1;
      }

      int error;
      if (FD_ISSET(sockfd, &rset) || FD_ISSET(sockfd, &wset)) {
          len = sizeof(error);
          if (getsockopt(sockfd, SOL_SOCKET, SO_ERROR, &error, &len) < 0) return -1;
      }
      
      done:
      if (error) {
          close(sockfd);
          errno = error;
          return -1;
      }
      return 0;
  
  结合以上代码，非阻塞的connect有一些坑在那，例如：RTT时间可能很短，瞬间就连上了，这种情况下不能在select里面等待；
  源自Berkeley的实现，在connect成功建立连接时，描述符可写，在connect连接出错时，描述符既可读又可写；
  没法判定连接是否成功（当连接成功，立马收到数据后，描述符也是可读又可写的，但是连接是没有错误发生的）。

  说到如何检测连接是否成功，网友们提出了各式各样的可行办法：
  以长度为0的参数调用read；
  再connect一次，这次必须失败并返回EISCONN错误；
  总之非阻塞的connect就是最蛋疼的做法啊，虽然它效率很高。

+ ##### **多进程的客户端** #####
  
  上面说道的非阻塞程序性能是好，缺点就是不那么容易正确的实现。
  这里要说到的多进程的实现模式，把读和写分为两个进程，效率略低于上面的非阻塞方式，
  但是代码却简单了很多。直接看代码吧：
  
      pid_t pid;
      char sendLine[MAX_LEN], recvLine[MAX_LEN];
      
      pid = fork();
      if (pid == 0) {//子进程 处理 服务器 到 标准输出
          while (readline(sockfd, recvLine, MAX_LEN) > 0) {
              fputs(recvLine, stdout);
          kill(getppid(), SIGTERM); //以防 父进程还在运行
          exit(0);  
      }

      //父进程
      while (fgets(sendLine, MAX_LEN, fp) != NULL) {
          writen(sockfd, sendLine, strlen(sendLine));
      }
      shutdown(sockfd, SHUT_WR); //读入了EOF
      pause();
      return;

  结合以上代码可以看出，这样实现确实是很简单。TCP链接是全双工的，父子进程共享一个套接字：父进程往里写，子进程往外读。
  运行原理见下图：

  ![multi-progress-client](/image/multi-progress-client.png)
  
+ ##### **多线程的客户端** #####
  
  看上去和多进程差不多，直接看代码吧。

      /*
      两个全局变量，在最外面声明，
      也可以用参数传递的方式向子线程传递参数
      这里为了简单起见，使用全局变量
      */
      static int sockfd;
      static FILE *fp;

      void str_cli() {
          char recvLine[MAX_LEN];
          pthread_t tid;
          pthread_create(&tid, NULL, copyto, NULL);
          while (readline(sockfd, recvLine, MAX_LEN) > 0) {
            fputs(recvLine, stdout);
          }
      }
      
      void* copyto(void *arg) {
          char sendLine[MAX_LEN];
          while (fgets(sendLine, MAX_LEN, fp) != NULL) {
              writen(sockfd, sendLine, strlen(sendline));
          }
          shutdown(sockfd, SHUT_WR);
          return NULL;
      }

+ ##### **总结** #####
  
  几种设计方法的速度对比起来，停等版本的客户端速度最慢，
  I/O复用的版本比停等的版本快30倍，
  非阻塞的版本比I/O复用的版本快1倍，
  多线程（多进程）版本速度略逊于非阻塞版本。
  但是考虑到多线程（多进程）的实现简单程度远高于非阻塞版本，比较推荐的是多线程（多进程）的实现方式。

#### **服务器程序设计范式** #####

+ ##### **迭代的服务器** #####

      while (1) ｛
          connfd = accept(listenfd, (SA*) NULL, NULL);
          read(connfd, ...);
          writen(connfd, ...);
          close(connfd);
      ｝

  迭代的服务器就是这么酷炫，完全搞定了一个客户才转向下一个。它在这里存在的意义就是为了计算出多线程（多进程）服务器，
  在进程（线程）控制上，所消耗的cpu时间。因为迭代的服务器可以认为是不需要进程（线程）控制的，
  所以它的时间就完完全全是收发数据的时间。
  多线程（多进程）版本的程序，减掉迭代服务器的时间，就是用于进程（线程）控制的时间。
  注：在客户不断到来的情况下才有意义，这样就没有在accept上阻塞的时间。

+ ##### **TCP并发服务器，每个客户一个子进程** #####
  
      while (1) ｛
          connfd = accept(listenfd, (SA*) NULL, NULL);

          if (fork() == 0) { //子进程
              close(listenfd);

              //读写套接字
              //处理数据      
     
              exit(0);
          }
          close(connfd);
      ｝
  
  大致框架还是很清晰的，核心思想就是，来了一个客户，就为它创建一个进程来处理。
  这种设计方法的缺陷在于为每个客户临时fork子进程比较消耗cpu时间，特别是对于现在繁忙的web服务器，每天的连接数那么惊人，
  所以还要用各种方法来改进。

+ ##### **TCP预先分配子进程服务器，accept无上锁保护** #####
  
  这种方法就是，一开始fork多个进程，每个进程都调用accept，然后等待客户的到达。
  这么做的好处是新用户到达时，无需承担fork的开销。但是也有一些值得关注的点：
  
  1. 需要增加一些代码来应对客户负载的变动，
     当剩余可用子进程数低于阀值时，应该创建更多的子进程来等待客户的到达；
     当剩余可用子进程数高于阀值时，就应该终止一些子进程。
  
  2. N个子进程在同一个监听描述符上调用accept，
     当客户到达时，N个子进程均被唤醒（因为他们调用accept等待的是一个描述符，所以在同一个等待队列上排队），
     但是只有第一个子进程会获得那个客户，其余的N-1个子进程继续睡眠，这被称为惊群（thundering herd）效应。
     当N过大时，惊群效应带来的性能损失显著。

  3. 引发select冲突。多个进程引用一个描述符套接字，在其上调用select，等待监听套接字变为可读而不是阻塞在accept上，
     新用户到达时，所有阻塞在select上的进程都会被惊醒。
     因为在socket结构中为存放本套接字就绪时应唤醒的进程分配的仅仅是一个进程ID的空间，如果有多个进程等待一个套接字，
     必须全部唤醒，因为不知道哪些进程受刚就绪的套接字的影响。
     可以认为这是一种在select上的惊群吧，这又增加了额外的开销。

+ ##### **TCP预先分配子进程服务器，accept上锁保护** #####

  为了解决上一个服务器中总是出现的惊群效应，应该在accept的前后放置锁，
  使得只有一个子进程阻塞在accept上，其他子进程在等候锁。
  锁可以用文件锁也可以用线程锁，文件锁的特点是容易移植，但是涉及文件系统速度慢；
  线程锁需要具备不同进程之间的线程上锁条件，但是速度快。
  不同进程使用线程上锁要求：1.互斥锁变量必须存放于进程共享的内存区；2.必须告知线程函数库这是不同进程之间共享的互斥锁。
  所以需要先在进程之间创建一个共享的内存区。例如：

      pthread_mutexattr_t mattr;
      fd = open("/dev/zero", O_RDWR, 0);
      mptr = mmap(0, sizeof(pthread_mutex_t), PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
      close(fd);

      pthread_mutexattr_init(&mattr);
      pthread_mutexattr_setpshared(&mattr, PTHREAD_PROCESS_SHARED);
      pthread_mutex_init(mptr, &mattr);

+ ##### **TCP预先分配子进程服务器，传递描述符** #####

  这个版本的服务器只在父进程中调用accept，然后以某种形式把描述符传递给空闲子进程，从而不用在accept上锁了。
  但是这种方式下，父进程需要读取子进程的闲忙状况，需要向子进程传递描述符，涉及到进程间的通信，所以多少有些复杂。
  进程间通信又是个很大的话题了，详见该书的30.9节吧。

  但是值得注意的是，前几种方式，子进程获得客户都是由操作系统调度的，而这种，相当于是我们自己管理的子进程调度。
  那么各个子进程获得新客户的几率，就取决于自己的实现了。

+ ##### **TCP多线程服务器** #####
  
  和多进程差不多吧，也是可以新用户到达时现场创建线程、预先分配子线程然后各自accept、预先分配子线程然后由主线程统一accept。
  不过通常来说使用线程快于使用进程，如果操作系统支持线程的话。线程这东西，共享变量记得上锁之类的吧，需要注意的。

#### **题外话** #####
  
  写到这就差不多了，感觉这本书除了讲述网络编程相关的知识，还让我了解了很多我觉得应该是在**`APUE`**中才应该出现的知识。
  现在理解起来，TCP就像是在机器上一直运行的程序，
  我们需要发数据时调用send，只是把数据拷贝到TCP的发送缓冲区而已，根本不能保证对面收到。
  说白了，send和recv只是做字节拷贝的工作。真正的和对端通信，是由那个在你机器上一直运行的TCP去做的。
  感觉这比喻不是那么恰当，但是**`“send和recv只是做字节拷贝的工作”`**确实让我很震撼，一定是我知道的太少了，我一定不是最后一个知道的。
