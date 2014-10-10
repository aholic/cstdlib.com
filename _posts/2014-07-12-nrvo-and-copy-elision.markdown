---
layout: post
title:  "编译器关于临时对象的优化"
date:   2014-07-12 08:42:35
categories: jekyll update
---

我当C++助教的时候，曾今看过这么一段代码：

{% highlight c++%}
map<string, int> GetWordCounter(string fileName) {
    map<string, int> wordCounter;
    //read file 
    //fill the map
    return wordCounter;
}

vector<pair<string, int> > GetTopWords(map<string, int> wordCounter, int top) {
    vector<pair<string, int> > topWords;
    // find words with max number of occurrences
    // ...
    return topWords;
}

//call the function GetTopWords to find top100 words
vector<pair<string, int> > topWords = GetTopWords(GetWordCounter("input.txt"), 100);
{% endhighlight%}

当时我一看这代码就很瞎，然后我就告诉他：

> **`GetWordCounter`**函数的返回值设计的有问题，
> wordCounter从函数里面返回的时候，是拷贝构造出来的，
> 当这个wordCounter类占用很大内存的时候，尤其显得不合适。
> 
> 另外，**`GetTopWords`**函数的参数设计也有问题，因为在传入GetTopWords内部时，
> 参数又要拷贝构造一份才能传递到函数里面。
> 
> 解决这两个问题比较好的方式是传递引用的方式。

当时为了更好的说明我的观点，我在`GetTopWords`内部加上了断点。
我想单步调试运行到该断点，比较进入`GetTopWords`函数前后程序占用内存的差异。
因为整个文件读下来，`wordCounter`的大小应该有好几M，
那么进入`GetTopWords`函数后，占用的内存至少比之前要多好几M。
正想见证一下奇迹，说明一下我的观点是多么的正确，可是当时的实验结果竟然是，占用的内存没变化！
我很郁闷，只能说：“这不科学”。后来也忘了去查查为什么。


直到前段时间看C++11的移动语义和右值引用的时候，顺便看到了编译器的一些优化措施，让我觉得茅塞顿开。
我只想和当时的那个学生说一句！“再来让我给你讲一遍！T.T”


在C++11出来之前，C++还不支持移动语义，那临时对象的效率问题就让人很头疼。
但是情况也没有想象的那么糟糕，因为编译器能优化其中的部分问题。
下面主要介绍一下**`(N)RVO`**和**`Copy Elision`**。

#### **(N)RVO** ####
(N)RVO全称是**`(Named)Return Value Optimization`**，即**`（具名）返回值优化`**。
RVO指的是编译器让调用函数在其栈上分配空间，然后将这块内存的地址传递给被调函数，被调函数直接在这块内存上构造返回值。
这样就消除从函数内部return出来时临时对象的复制问题。例如如下代码：

{% highlight c++%}
struct Foo {
    Foo(int) { cout << "ctr" << endl; }
    Foo(const Foo& foo) { cout << "cp" << endl; }
};
Foo getFoo() {return Foo(1024);}
Foo foo = getFoo();
{% endhighlight %}

按常理来说，在`getFoo`函数体内部的表达式`Foo(1024)`中会构造一份Foo临时对象，
然后将这个临时对象拷贝到返回值返回值存储区，
然后从返回值存储区拷贝到`foo`的存储区。也就是需要1次构造，2次复制。
而有了返回值优化，只需要1次构造，无需拷贝，因为他直接在`foo`的存储区构造了这个对象。
要实验的话，可以用以上代码，g++(clang++)编译的时候用**`-fno-elide-constructors`**选项来控制是否开启优化。

而NRVO指的是如下代码，也能被以类似的方式优化：

{% highlight c++%}
struct Foo {
    Foo(int) { cout << "ctr" << endl; }
    Foo(const Foo& foo) { cout << "cp" << endl; }
};
Foo getFoo() {Foo foo(1024); return foo;}
Foo foo = getFoo();
{% endhighlight %}

#### **Copy Elision** ####
Copy Elision，即**`复制省略`**。
复制省略指的是，当函数参数以值传递的方式传入函数内部时，通常要求建立一份参数的拷贝。
当传入的参数是右值时，则无需建立这样一份拷贝，直接使用源对象即可。例如如下代码：

{% highlight c++%}
struct Foo {
    Foo(int) { cout << "ctr" << endl; }
    Foo(const Foo& foo) { cout << "cp" << endl; }
};
void useFoo(Foo foo) {}
useFoo(Foo(10));
{% endhighlight %}

理论上来说，在调用`useFoo(Foo(10))`时，
先由`Foo(10)`来构造一个Foo对象，
然后把这个`Foo`对象拷贝到函数内部。也就是需要1次构造，1次复制。
而由于和(N)RVO类似的原理，有了Copy Elision，只需要1次构造。

#### **写在最后** ####
(N)RVO和Copy Elision都只能解决部分临时对象的问题。
例如以下代码，Copy Elision便无能为力了：

{% highlight c++%}
struct Foo {
    Foo(int) { cout << "ctr" << endl; }
    Foo(const Foo& foo) { cout << "cp" << endl; }
};
void useFoo(Foo foo) {}

Foo foo(10);
useFoo(foo);
{% endhighlight %}

在上面代码中，将`foo`传入函数`useFoo`时，复制省略无法被启用，
因为源对象以后还会被用到，所以不能直接使用源对象。
为了彻底的解决临时对象效率的问题，请看[C++11的移动语义和右值引用]({{ site.url }}/jekyll/update/2014/07/12/new-features-in-c++11/)。


另一方面，由于绝大多数（也许99.9%）编译器都支持(N)RVO和Copy Elision，
当需要在函数内部复制构造参数时，传递引用然后拷贝构造是很愚蠢的做法，直接传值是更好的做法。
因为直接传值的话，编译器能利用一些优化措施优化自然更好，即使不能优化，也不会更糟。
为此，大神[Dave Abrahams](http://en.wikipedia.org/wiki/David_Abrahams_(computer_programmer))
写下了著名的[want spped? pass by value](http://fpcfjf.blog.163.com/blog/static/5546979320133174350249/)
