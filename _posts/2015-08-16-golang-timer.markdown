---
layout: post
title:  "Go语言中可取消的定时器"
date:   2015-08-16 21:50:00
categories: tech
keywords: "go, golang, timer, cancel"
description: "本文介绍了如何在Go语言中实现可取消的定时器"
---

#### **需求** ####
需求，最重要的就是需求，需求就是要有一个定时器，能定时的做某件事，并且在等待该时刻到来的途中，能把这个定时器关掉，即支持**`cancel`**操作。

#### **标准库的Timer/Ticker** ####
标准库提供里Timer类和Ticker类，前者是定时做某事（仅一次），后者是有周期的做某事（一直做）。

常见用法：

    t := time.NewTimer(3 * time.Second)
    done := make(chan bool, 1)

    go func() {
        <-t.C
        fmt.Println("after 3 seconds")
        done <- true
    }()

    <-done
    
但是注意到这个Timer对象t是无法关闭的，好的，你可能会发现Timer类提供了Stop方法。但是你也会发现，这个方法并没有什么卵用。
不如我们试试这段程序：

    t := time.NewTimer(3 * time.Second)
    done := make(chan bool, 1)

    go func() {
        <-t.C
        fmt.Println("after 3 seconds")
        done <- true
    }()
        
    t.Stop()
    <-done
    
直觉上来看，那个新起的goroutine在等待的过程中，主线程会把定时器关掉，然后直接输出“after 3 seconds”，然后程序皆大欢喜的结束，然而显示会打烂你的脸，输出会是这样：

    $go run main.go 
    fatal error: all goroutines are asleep - deadlock!

    goroutine 1 [chan receive]:
    main.main()
        /home/troy/golang/timer/main.go:19 +0x156

    goroutine 6 [chan receive]:
    main.func·001()
        /home/troy/golang/timer/main.go:13 +0x53
    created by main.main
        /home/troy/golang/timer/main.go:16 +0x11d
    exit status 2
    
是的，程序死锁了。Stop并没有我们想要的功能，呵呵呵。还有人会说，调用close(t.C)就可以了，是的，可以，然后你回得到**`cannot close receive-only channel`**，
t.C是一个只读队列，无法调用close。

#### **怎么办** ####
其实这是个挺常见的需求吧，直接贴代码，加一个管道用select

    func startTimer(f func(now time.Time)) chan bool {
        done := make(chan bool, 1)
	      go func() {
		        t := time.NewTimer(time.Second * 3)
		        defer t.Stop()
		        for {
			          select {
		    	      case now := <-t.C:
				            f(now)
			          case <-done:
				            fmt.Println("done")
				            return
                }
		        }
	      }()
	      return done
    }

	  done := startTimer(func(now time.Time) {
        fmt.Println(now)
	  })
	  time.Sleep(1 * time.Second)
	  close(done)
    
select的其他实用的用法看看[这里](http://yanyiwu.com/work/2014/11/08/golang-select-typical-usage.html)