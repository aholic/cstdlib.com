---
layout: post
title:  "C++11中的新特性"
date:   2014-07-12 02:56:34
categories: jekyll update
---

前段时间看完了`《STL源码剖析》`，觉得对C++又有了更多的认识和理解。

    //assume v is vector of int
    vector<int> v;

    //find max element in vector<int>
    int maxNum = MIN_INT;
    for (size_t i = 0; i < v.size(); i++) {
        if (maxNum < v[i]) {
            maxNum = v[i];
        }
    }

    //or use STL do it
    vector<int>::iterator maxNum = max_element(v.begin()， v.end());

以前我觉得我一定会写出以上代码中的第一种方法。
可是对比第二种方法，优雅程度高下立判。
**`在可以用STL的地方尽可能的正确使用STL。`**
想起查资料经常查到[stackoverflow](http://stackoverflow.com)，
看到别人写的C++代码中，总是有一些让我惊呼“我擦还可以这样”的一些C++的特性或者是STL函数。
很明显，这是因为我对C++的撑握程度和理解水平太低了。
为了紧跟时代的步伐，我决定跟进C++11，实际上C++11已经不是很新了，是一个11年就出来了的标准了。
于是我看了一本书`《深入理解C++11》`，写这篇文章主要说一说我对右值引用和移动语义的理解吧。

### **移动语义（move semantics）** ###
先说一下拷贝的语义，拷贝指的是在源对象不变的情况下，复制出一个新的对象。
而移动的语义呢？移动指的是将源对象的资源“窃取过来”，完成资源所有权的转移。
也就是说，在移动之后，源对象的资源没有了，被转移到新的对象里面了。
举个具体的例子，一个正常的类，拷贝构造函数会做两件事：1.申请地址空间；2.将资源复制过来。

    struct Foo {
        char* _data;
        size_t _length;
        Foo(const Foo& obj) {
            _length = obj._length;
            _data = new char[obj._length + 1];
            std::copy(obj._data, obj._data + _length, _data);
        }
    };

但是，当obj是一个临时对象时，即obj不会在被其他地方用到，或者说obj的生命周期要结束了，
那这次复制就显得很浪费，因为可以直接把它从obj中“移动过来”。
也就是说，需要这么一个构造函数：

    Foo(dying Foo& obj) {
        _length = obj._length;
        _data = obj._data;
        obj._data = nullptr;
    }

当然，dying不是一个C++关键字，只是为了表示obj的生命周期即将结束，或者说是`“将亡”`。
如果有了这么一个构造函数来处理临时对象的问题，那么C++中广为诟病的临时对象效率问题将会被很好的解决。
所以，需要做的，就是判断参数到底是左值还是右值，对于右值，就可以使用这个`“移动构造函数”`。
为了区分参数的类型，传统的办法是使用重载机制，但是仅仅是重载机制不够解决这些问题，
还需要巧妙地利用模版类型推导机制，需要掺杂一些复杂的东西。
碉堡的C++天才[Andrei Alexandrescu](http://en.wikipedia.org/wiki/Andrei_Alexandrescu)为了解决这个问题，提出了[MOJO](http://www.drdobbs.com/move-constructors/184403855)。
虽然是天才的解决办法，但是，他的解决办法始终有点复杂（对我而言）。
回顾一下移动语义的初衷和解决方案：
**`为了解决临时对象拷贝带来的效率低下，需要实现移动语义；实现移动语义的前提是能够识别参数中的右值类型。`**
于是，有了**`右值引用`**。

### **右值引用（rvalue reference）** ###
右值的具体定义就不介绍了，简单的来说就是`“不能对它取地址的值”`。
C++中加入了一个新的引用类型——右值引用。他的语法是**`&&`**。突出特点是可以用它来绑定到右值。
有了右值引用，前文提到的`“移动构造函数”`可以被很方便的实现：

    Foo(Foo&& obj) {
        _length = obj._length;
        _data = obj._data;
        obj._data = nullptr;
    }

为了产生右值，C++11的标准库提供了**`std::move`**，它的作用很简单，把他的参数作为一个右值返回。
move容易被误用，例如以下代码：

    #include <iostream>
    #include <cstring>

    using namespace std;

    struct Foo {
        int* _data;
        Foo(int data) : _data(new int(data)) {}
        Foo(Foo&& obj) : _data(obj._data) {
            obj._data = nullptr;
        }
    };

    int main() {
        Foo foo1(100);
        Foo foo2(move(foo1));
        cout << *(foo1._data) << endl;
        return 0;
    }

上面的代码运行时会出现`segment fault`，因为`foo1._data`已经在移动构造函数中被置为了`nullptr`。
这个错误的根源是一个对象的生命周期还没结束，就把它的资源“move”给别的对象了。
所以当需要使用move时，应该明确这个对象的生命周期要结束了，或者是这个对象以后不再被使用了。
接下来再看一个正确使用**`std::move`**的例子：

    #include <iostream>
    using namespace std;

    struct Resource {
        int *_data;
        int _sz;
        Resource(int sz) : sz(sz), _data(new int[sz]) {}
        ~Resource() {delete [] _data;}
        Resource(Resource&& res) : _sz(res._sz), _data(res._data) {
            res._data = nullptr;
        }
    };

    struct Foo {
        Resource _res;
        Foo(int sz) : _res(sz) {}
        Foo(Foo&& foo) : _res(move(foo._res)) {}
    };

    int main() {
        Foo foo1(1024);
        Foo foo2(foo1);
        return 0;
    }

在以上代码中定义了Foo和Res两个类，Foo包含了Res类。
在Foo的移动构造函数中，为了调用了成员\_res的移动构造函数，对其使用了**`std::move`**。
这里之所以正确的使用了move，是因为foo.\_res是foo的成员，既然foo的生命周期要结束了（因为foo是一个右值引用），
那么foo.\_res的生命周期也要结束了。

### **总结** ##
以上两节的标题和内容其实有点奇怪，
在**`移动语义`**中介绍的是右值引用的由来，
而在**`右值引用`**中介绍的却是如何实现移动语义。
但是细细想来也不觉得奇怪，
主要抓住的重点应该在于**`右值引用之所以出现，就是为了解决移动语义的问题`**，
另外，右值引用还解决了**`完美转发`**的问题，以后有时间再谈吧。

### **题外话** ##
再顺便说说**`(N)RVO`**和**`Copy Elision`**吧。
主要是感觉他们和临时对象有点关系。毕竟，在移动语义出来之前，
C++标准组因为临时变量的事被喷了很久了，所以在标准中不惜代价的允许(N)RVO。

##### **(N)RVO** #####
(N)RVO全称是**`(Named)Return Value Optimization`**，即**`（具名）返回值优化`**。
RVO指的是编译器让调用函数在其栈上分配空间，然后将这块内存的地址传递给被调函数，被调函数直接在这块内存上构造返回值。
这样就消除从函数内部return出来时临时对象的复制问题。例如如下代码：

    struct Foo {
        Foo(int) { cout << "ctr" << endl; }
        Foo(const Foo& foo) { cout << "cp" << endl; }
    };
    Foo getFoo() {return Foo(1024);}
    Foo foo = getFoo();

按常理来说，在`getFoo`函数体内部的表达式`Foo(1024)`中会构造一份Foo临时对象，
然后将这个临时对象拷贝到返回值返回值存储区，
然后从返回值存储区拷贝到foo的存储区。也就是需要1次构造，2次复制。
而有了返回值优化，只需要1次构造，无需拷贝，因为他直接在foo的存储区构造了这个对象。
要实验的话，可以用以上代码，编译的时候由**`-fno-elide-constructors`**选项来控制是否开启优化。

而NRVO指的是如下代码，也能被以类似的方式优化：

    struct Foo {
        Foo(int) { cout << "ctr" << endl; }
        Foo(const Foo& foo) { cout << "cp" << endl; }
    };
    Foo getFoo() {Foo foo(1024); return foo;}
    Foo foo = getFoo();

##### **`Copy Elision`** #####
Copy Elision，即复制省略。
复制省略指的是，当函数参数以值传递的方式传入函数内部时，通常要求建立一份参数的拷贝。
当传入的参数是右值时，则无需建立这样一份拷贝，直接使用源对象即可。例如如下代码：

    struct Foo {
        Foo(int) { cout << "ctr" << endl; }
        Foo(const Foo& foo) { cout << "cp" << endl; }
    };
    void useFoo(Foo foo) {}
    useFoo(Foo(10));

理论上来说，在调用`useFoo(Foo(10))`时，
先由`Foo(10)`来构造一个Foo对象，
然后把这个Foo对象拷贝到函数内部。也就是需要1次构造，1次复制。
而由于和(N)RVO类似的原理，有了Copy Elision，只需要1次构造。

但是(N)RVO和Copy Elision都只能解决部分临时对象的问题，有些问题还是得用移动语义来解决的。
由于绝大多数（也许99.9%）编译器都支持(N)RVO和Copy Elision，
当需要在函数内部参数时，传递引用然后拷贝构造是很愚蠢的做法，直接传值是更好的做法。
为此，大神[Dave Abrahams](http://en.wikipedia.org/wiki/David_Abrahams_(computer_programmer))
写下了著名的[want spped? pass by value](http://fpcfjf.blog.163.com/blog/static/5546979320133174350249/)

### **相关阅读** ###
\[1\]：[译：详解C++右值引用](http://jxq.me/2012/06/06/%E8%AF%91%E8%AF%A6%E8%A7%A3c%E5%8F%B3%E5%80%BC%E5%BC%95%E7%94%A8/#section_06)    
\[2\]：[我家的返回值才不可能这么傲娇（右值引用和移动语义）](http://darkc.at/cxx_rvalue_reference/)    
\[3\]：[《C++0x漫谈》系列之：右值引用(或“move语意与完美转发”)(上)](http://blog.csdn.net/pongba/article/details/1684519)

