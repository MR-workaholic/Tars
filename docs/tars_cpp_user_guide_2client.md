# 2. C++客户端

客户端可以不用写任何协议通信代码即可完成远程调用。客户端代码同样需要包含hello.h文件。
## 2.1. 通信器

完成服务端以后，客户端需要对服务端完成收发包的工作。客户端对服务端完成收发包的操作是通过通信器（communicator）来实现的。

** 注意:一个Tars服务只能使用一个Communicator，用Application::getCommunicator()获取即可（若不是Tars服务，要使用通讯器，则自己创建，也只能一个）。

通信器是客户端的资源载体，包含了一组资源，用于收发包、状态统计等功能。

通信器的初始化如下：
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//用配置文件初始化通信器
c-> setProperty(conf);
//直接用属性初始化
c->setProperty("property", "tars.tarsproperty.PropertyObj");
c->setProperty("locator", "tars.tarsregistry.QueryObj@tcp -h ... -p ...");
```
说明：
> * 通信器的配置文件格式后续会介绍；
> * 通信器缺省不采用配置文件也可以使用，所有参数都有默认值；
> * 通信器也可以直接通过属性来完成初始化；
> * 如果需要通过名字来获取客户端调用代理，则必须设置locator参数；

通信器属性说明：
> * locator: registry服务的地址，必须是有ip port的，如果不需要registry来定位服务，则不需要配置；
> * sync-invoke-timeout：调用最大超时时间（同步），毫秒，没有配置缺省为3000
> * async-invoke-timeout：调用最大超时时间（异步），毫秒，没有配置缺省为5000
> * refresh-endpoint-interval：定时去registry刷新配置的时间间隔，毫秒，没有配置缺省为1分钟
> * stat：模块间调用服务的地址，如果没有配置，则上报的数据直接丢弃；
> * property：属性上报地址，如果没有配置，则上报的数据直接丢弃；
> * report-interval：上报给stat/property的时间间隔，默认为60000毫秒；
> * asyncthread：异步调用时，回调线程的个数，默认为1；
> * modulename：模块名称，默认为可执行程序名称；

通信器配置文件格式如下：
```
<tars>
  <application>
    #proxy需要的配置
    <client>
        #地址
        locator                     = tars.tarsregistry.QueryObj@tcp -h 127.0.0.1 -p 17890
        #同步最大超时时间(毫秒)
        sync-invoke-timeout         = 3000
        #异步最大超时时间(毫秒)
        async-invoke-timeout        = 5000
        #刷新端口时间间隔(毫秒)
        refresh-endpoint-interval   = 60000
        #模块间调用
        stat                        = tars.tarsstat.StatObj
        #属性上报地址
        property                    = tars.tarsproperty.PropertyObj
        #report time interval
        report-interval             = 60000
        #网络异步回调线程个数
        asyncthread                 = 3
        #模块名称
        modulename                  = Test.HelloServer
    </client>
  </application>
</tars>
```
使用说明：
> * 当使用Tars框架做服务端使用时，通信器不要自己创建，直接采用服务框架中的通信器就可以了，例如：Application::getCommunicator()->stringToProxy(...)，对于纯客户端情形，则要用户自己定义个通信器并生成代理（proxy）；
> * getCommunicator()是框架Application类的静态函数，随时可以获取；
> * 对于通信器创建出来的代理，也不需要每次使用的时候都stringToProxy，初始化时建立好，后面直接使用就可以了；
> * 代理的创建和使用，请参见下面几节；
> * 对同一个Obj名称，多次调用stringToProxy返回的Proxy其实是一个，多线程调用是安全的，且不会影响性能；
> * 可以通过Application::getCommunicator()->getEndpoint（”obj”）获取obj对应ip列表。另外一种获取方式为直接从通信器生成的proxy获取如Application::getCommunicator()->stringToProxy(…) ->getEndpoint()

## 2.2. 超时控制

超时控制是对客户端proxy（代理）而言。上节中通信器的配置文件中有记录：
```cpp
#同步最大超时时间(毫秒)
sync-invoke-timeout          = 3000
#异步最大超时时间(毫秒)
async-invoke-timeout         = 5000
```
上面的超时时间对通信器生成的所有proxy都有效。

如果需要单独设置超时时间，设置如下：

针对proxy设置超时时间：
```cpp
ProxyPrx  pproxy;
//设置该代理的同步的调用超时时间 单位毫秒
pproxy->tars_timeout(3000);
//设置该代理的异步的调用超时时间 单位毫秒
pproxy->tars_async_timeout(4000);
```

针对接口设置超时时间：
```cpp
//设置该代理的本次接口调用的超时时间.单位是毫秒，设置单次有效
pproxy->tars_set_timeout(2000)->a(); 
```

## 2.3. 调用

本节会详细阐述tars的远程调用的方式。

首先简述tars客户端的寻址方式，其次会介绍客户端的调用方式，包括但不限于单向调用、同步调用、异步调用、hash调用等。

### 2.3.1. 寻址方式简介

Tars服务的寻址方式通常可以分为如下两种方式，即服务名在主控注册和不在主控注册，主控是指专用于注册服务节点信息的名字服务（路由服务）。
名字服务中的服务名添加则是通过操作管理平台实现的。

对于没有在主控注册的服务，可以归为直接寻址方式，即在服务的obj后面指定要访问的ip地址。客户端在调用的时候需要指定HelloObj对象的具体地址：

即：Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985

Test.HelloServer.HelloObj：对象名称

tcp：tcp协议

-h：指定主机地址，这里是127.0.0.1 

-p：端口地址，这里是9985

如果HelloServer在两台服务器上运行，则HelloPrx初始化方式如下：
```cpp
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985:tcp -h 192.168.1.1 -p 9983");
```
即，HelloObj的地址设置为两台服务器的地址。此时请求会分发到两台服务器上（分发方式可以指定，这里不做介绍），如果一台服务器down，则自动将请求分到另外一台，并定时重试开始down的那一台服务器。

对于在主控中注册的服务，服务的寻址方式是基于服务名进行的，客户端在请求服务端的时候则不需要指定HelloServer的具体地址，但是需要在生成通信器或初始化通信器的时候指定registry(主控中心)的地址。

这里指定registry的地址通过设置通信器的参数实现，如下：
```
CommunicatorPtr c = new Communicator();
c->setProperty("locator", "tars.tarsregistry.QueryObj@tcp -h .. -p ..")
```
由于依赖registry的地址，因此registry必须也能够容错，这里registry的容错方式与上面中一样，指定两个registry的地址即可。

### 2.3.2. 单向调用

所谓单向调用，表示客户端只管发送数据，而不接收服务端的响应，也不管服务端是否接收到请求。
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//用配置文件初始化通信器
c-> setProperty(conf);
//生成客户端代理
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985");
//发起远程调用
string s = "hello word";
string r;
pPrx->async_testHello(NULL, s);
```
### 2.3.3. 同步调用

请看如下调用示例：
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//用配置文件初始化通信器
c-> setProperty(conf);
//生成客户端代理
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985");
//发起远程同步调用
string s = "hello word";
string r;
int ret = pPrx->testHello(s, r);
assert(ret == 0);
assert(s == r);
```
上述例子中表示：客户端向HelloServer的HelloObj对象发起远程同步调用。

### 2.3.4. 异步调用

定义异步回调对象：
```cpp
struct HelloCallback : public HelloPrxCallback
{
//回调函数
virtual void callback_testHello(int ret, const string &r)
{
    assert(r == "hello word");
}

virtual void callback_testHello_exception(tars::Int32 ret)
{
    assert(ret == 0);
    cout << "callback exception:" << ret << endl;
}
};

TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//用配置文件初始化通信器
c-> setProperty(conf);
//生成客户端代理
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985");
//定义远程回调对象
HelloPrxCallbackPtr cb = new HelloCallback;

//发起远程调用
string s = "hello word";
string r;
pPrx->async_testHello(cb, s);
```
注意：
> * 当接收到服务端返回时，HelloPrxCallback的callback_testHello会被响应。
> * 如果调用返回异常或超时，则callback_testHello_exception会被调用，ret的值定义如下：


```cpp
//定义TARS服务给出的返回码
const int TARSSERVERSUCCESS       = 0;       //服务器端处理成功
const int TARSSERVERDECODEERR     = -1;      //服务器端解码异常
const int TARSSERVERENCODEERR     = -2;      //服务器端编码异常
const int TARSSERVERNOFUNCERR     = -3;      //服务器端没有该函数
const int TARSSERVERNOSERVANTERR  = -4;      //服务器端没有该Servant对象
const int TARSSERVERRESETGRID     = -5;      //服务器端灰度状态不一致
const int TARSSERVERQUEUETIMEOUT  = -6;      //服务器队列超过限制
const int TARSASYNCCALLTIMEOUT    = -7;      //异步调用超时
const int TARSINVOKETIMEOUT       = -7;      //调用超时
const int TARSPROXYCONNECTERR     = -8;      //proxy链接异常
const int TARSSERVEROVERLOAD      = -9;      //服务器端超负载,超过队列长度
const int TARSADAPTERNULL         = -10;     //客户端选路为空，服务不存在或者所有服务down掉了
const int TARSINVOKEBYINVALIDESET = -11;     //客户端按set规则调用非法
const int TARSCLIENTDECODEERR     = -12;     //客户端解码异常
const int TARSSERVERUNKNOWNERR    = -99;     //服务器端位置异常
```

### 2.3.5. 指定set方式调用

目前框架已经支持业务按set方式进行部署，按set部署之后，各个业务之间的调用对业务开发来说是透明的。但是由于有些业务有特殊需求，需要在按set部署之后，客户端可以指定set名称来调用服务端，因此框架则按set部署的基础上增加了客户端可以指定set名称去调用业务服务的功能。

详细使用规则如下，

假设业务服务端HelloServer部署在两个set上，分别为Test.s.1和Test.n.1。那么客户端指定set方式调用如下：
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//用配置文件初始化通信器
c-> setProperty(conf);
//生成客户端代理
HelloPrx pPrx_Tests1 = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985","Test.s.1");

HelloPrx pPrx_Testn1 = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985","Test.n.1");

//发起远程调用
string s = "hello word";
string r;

int ret = pPrx_Tests1->testHello(s, r);

int ret = pPrx_Testn1->testHello(s, r);
```
注意：
> * 指定set调用的优先级高于客户端和服务端自身启用set的优先级，例如，客户端和服务端都启用了Test.s.1，如果客户端创建服务端代理实例时指定了Test.n.1，那么实际请求就会发送到Test.n.1的节点上（前提是这个set有部署服务端）。
> * 创建的代理实例也是只需调用一次

### 2.3.6. Hash调用

由于服务可以部署多台，请求也是随机分发到服务上，但是在某些场合下，希望某些请求总是在某一台服务器上，对于这种情况Tars提供了简单的方式实现：

假如有一个根据QQ号查询资料的请求，如下：
```cpp
QQInfo qi = pPrx->query(uin);
```
通常情况下，同一个uin调用，每次请求落在的服务器地址都不一定，但是采用如下调用：
```cpp
QQInfo qi = pPrx->tars_hash(uin)->query(uin);
```
则可以保证，每次uin的请求都在某一台服务器。

注意：
> * 这种方式并不严格，如果后台某台服务器down，则这些请求会迁移到其他服务器，当它正常后，会迁移回来。
> * Hash参数必须是int，对于string来说，Tars基础库(util目录下)也提供了方法：tars::hash<string>()("abc")，具体请参见util/tc_hash_fun.h