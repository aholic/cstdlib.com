---
layout: post
title:  "如何使用PHPUnit测试Yaf的控制层"
date:   2015-05-23 13:58:32
categories: jekyll update
---

#### **背景** #####

没有背景的故事不是好故事。最近在帮一个同学写APP的后台，后台的需求比较简单，接收http请求，增/删/改/查数据库，然后撸一串json回去。我是用PHP写的后台，用的是[Yaf](http://www.laruence.com/manual/)框架，Yaf是鸟哥用C语言写的一个PHP开发框架，特别简洁，基本没有多余的功能。代码目前部署在sae上，得益于鸟哥在新浪工作，sae是直接支持Yaf的（PHP5.3运行环境支持，PHP 5.6运行环境不支持）。由于需求本身不复杂，所以很快的就累积了一些完成基本功能的代码，基本算是小小的demo有了吧。回过头来看看，想对代码的某些部分进行重构，但又怕构完了之后引入新的bug，所以必须引入一个测试的框架，我要做的测试也比较简单，就是把我所有的对客户端提供的接口都测一遍，这样基本也就够了。我用的是[PHPUnit](https://phpunit.de/)，用起来也算是方便的。

其实很久以前，给老赵做的那个网站是有用到PHPUnit来做单元测试的，当时项目刚开始做demo的时候，“雄心壮志”的添加了一些单元测试，但是非常可惜，随着代码数量不断膨胀，实在是工作量太大了，就放弃了。那时主要是用PHPUnit来做数据层的单元测试，也就是测试所有数据层提供的数据库封装接口（这里主要指的是例如getUserList这样的数据层接口，而不是再底层的PDO执行sql语句的那种接口）。

#### **测试数据层接口** ####

用PHPUnit做数据层的单元测试，这直观的来看是好理解的，分为以下几步：

1. 建立基境（将数据库中的数据设为初始状态）
2. 调用数据层的接口，例如getUserList，setPasswordByUserID
3. 断言预期的返回结果
4. 断言测试数据库中数据的状态
5. 销毁基境

可能稍微难理解一点的就是**`数据库中数据的状态`**吧，简单的来说，就是PHPUnit为我们提供了一些手段（PHP数组、XML、YAML等）来定义数据库中数据的状态。直接看个例子吧，就懂了，你一定就知道下面这个XML定义的数据库的状态是怎样的吧：

    <?xml version="1.0" ?>
    <dataset>
        <table name="user">
            <column>userID</column>
            <column>userPassword</column>
            <column>userNick</column>
            <row>
                <value>1</value>
                <value>xxxxxxx</value>
                <value>ahoLic</value>
            </row>
            <row>
                <value>2</value>
                <value>xxxxxxx</value>
                <value>kost</value>
            </row>
        </table>
    </dataset>

其实上面说做数据层接口的单元测试的过程是直观的好理解的，重要的原因是因为**`数据层的接口是可调用的`**，也就是说，在单元测试的case中，直接在代码里面调用数据层的接口就是了，数据层不再涉及我们写的代码以外的事了。换句话说，**`在真实项目中，谁调用了数据层接口`**？我们自己写控制层接口啊！既然在真实的项目中我们自己能调用，那么在单元测试的case中，我们也可以调用。

#### **测试控制层接口** ####

那么我现在在APP后台的这个项目中，要测试的是所有服务器向客户端提供的接口，也就是控制层接口，还是问问自己同样的问题：**`在真实项目中，谁调用了控制层接口`**？不知道，至少不是我写得代码去调用的。换句话说：控制层接口是不可被我们写的代码直接调用的。所以控制层接口的测试，一定是和框架有着紧密关系的。回想一下，实际环境中，服务器的处理流程是怎样的：

1. server程序接收到客户端发来的请求，解释执行相应的php文件
2. php文件解释执行，输出
3. 把输出装到http里面发回去

Yaf框架的控制层接口的执行是在以上的第2步，使用Yaf框架时，所有非文件请求全部指向一个index.php文件（这条规则定义在.htaccess文件中），然后Yaf框架根据路由规则和URI去执行相应的控制层接口。于是我们可以回答这个问题：**`在真实项目中，谁调用了控制层接口`**？Yaf框架；那么问题又来了：**`怎么调用的`**？不知道；所以由于我要测试控制层接口，我必须调用它们，我有两个办法：

1. 用curl等模拟不同的请求，断言请求的json结果。这其实是一种黑盒测试，不适合还要同时断言数据库状态的情况。
2. 了解Yaf框架调用控制层框架的过程，寻找开始处理一个请求的源头，从这个源头开始模拟。

好的，简单的了解了Yaf框架之后，发现它的源头在这：把一个**`Yaf_Request_Abstract`**交给**`Yaf_Dispatcher`**去dispatch，dispatch根据路由规则和**`Yaf_Request_Abstract`**中的信息，调用不同的控制层接口。所以我只要模拟不一样的**`Yaf_Request_Abstract`**出来就可以了，本质也是模拟不同的请求，但是这其实是一种白盒测试了，测试的东西已经涉及了代码内部了。

然后就会惊喜发现Yaf框架竟然有一种命令行的运行模式，这不就是为测试而生的吗？其实框架本身的手册里面也说了，为了方便测试，有一种命令行的运行模式。好吧，其实之前说了这么多，只是为了解释，为什么要这样才能测试。命令行运行，就是自己去创建一个**`Yaf_Request_Simple`**，这个类继承自**`Yaf_Request_Abstract`**。一个简单的例子如下：

    $request = new Yaf_Request_Simple("CLI", "Index", "User", "getUserInfo", array("userID" => 2));
    //the request equals to visit domain.com/Index/User/getUserInfo?userID=2
    $app = new Yaf_Application("conf.ini");
    $response = $app->getDispatcher()->returnResponse(true)->dispatch($request);
    assert($response->getBody(), json_encode(array('userID' => '2', 'userNick' => 'ahoLic')))

好的，花了很大的篇幅介绍数据层的测试和控制层的测试的区别，和控制层测试的问题是什么。接下来介绍在用PHPUnit来做Yaf的控制层测试，需要注意的问题。

#### **问题记录** ####

##### **Yaf框架要求运行时Yaf_Application是单例** #####
  
所以借用Yaf_Registry来实现这个单例

    class TestController extends PHPUnit_Framework_TestCase {
        private $__application = null;
        public function __construct() {
            $this->__application = Yaf_Registry::get('Application');
            if ($this->__application == null) {
                $this->__application = new Yaf_Application(ROOT_PATH."/conf/application.ini");
                Yaf_Registry::set('Application', $this->__application);
                Yaf_Registry::set('config', $this->__application->getConfig());
            }
        }
    }

##### **控制层接口怎么输出** #####

我以前的的控制层代码，输出总是很暴力的：

    //return {'userID' : '2', 'userNick' : 'ahoLic'} to client
    die(json_encode(array('userID' => '2', 'userNick' => 'ahoLic')));

真的很暴力，直接停止了当前进程，哈哈。目前用起来没有什么问题，但是有个潜在的问题，那就是万一框架拿到控制层接口还要包装一点什么的时候，就被你直接无情的略过了，以后一定不能再这么暴力了。用Yaf框架，要这么写：

    return $this->getResponse()->setBody(json_encode(array('userID' => '2', 'userNick' => 'ahoLic')));

关于这个不想解释得太多，因为我也没法解释，这个就是用Yaf这个框架就是这样的。

##### **yaf.use_spl_autoload** #####

php.ini里面记得加上：

    yaf.use_spl_autoload=1

主要影响的是当解释器找不到一个类时去哪找这个类，其实这个我不是很懂，感觉和PHPUnit的命名空间也有点关系，我是真的不懂PHP啊，特别是他的命名空间，真心不懂，也不想懂。另外拜托各位面试我的时候不要问我PHP是前端语言还是后端语言好么，问这样的问题真的好么？

#### 代码结构的组织在哪 ####

简单的代码组织结构可以看[这个](https://github.com/aholic/yaf-phpunit)

#### **相关资料** ####

\[1\]：[PHP: Yaf-Manual](http://php.net/manual/zh/book.yaf.php)    
\[2\]：[Yaf(Yet Another Framework)用户手册](http://www.laruence.com/manual/)    
\[3\]：[PHPUNIT单元测试YAF控制层](http://www.crackedzone.com/phpunit-yaf-controller.html)    
\[4\]：[PHPUnit手册](https://phpunit.de/manual/current/zh_cn/phpunit-book.html)    
