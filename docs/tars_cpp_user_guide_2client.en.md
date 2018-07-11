# 2. C++ Client

The client can complete the remote call without writing any code related to the protocol communication.The client code also needs to include the hello.h file.
## 2.1. Communicator

After the server is implemented, the client needs to send and receive data packets to the server. The client's operation of sending and receiving data packets to the server is implemented by Communicator.

** Note: A Tars service can only have one Communicator variable, which can be obtained with Application::getCommunicator() (if it is not the Tars service, create a communicator yourself).

The communicator is a carrier of client resources and contains a set of resources for sending and receiving packets, status statistics and other functions.

The communicator is initialized as follows:
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//Initialize the communicator with a configuration file
c-> setProperty(conf);
//Or initialize directly with attributes
c->setProperty("property", "tars.tarsproperty.PropertyObj");
c->setProperty("locator", "tars.tarsregistry.QueryObj@tcp -h ... -p ...");
```
Description:
> * The communicator's configuration file format will be described later.
> * Communicators can be configured without a configuration file, and all parameters have default values.
> * The communicator can also be initialized directly through the "Property Settings Interface".
> * If you need to get the RPC call proxy through the name server, you must set the locator parameter.

Communicator attribute description:
> * locator:The address of the registry service must be in the format "ip port". If you do not need the registry to locate the service, you do not need to configure this item.
> * sync-invoke-timeout:The maximum timeout (in milliseconds) for synchronous calls. The default value for this configuration is 3000.
> * async-invoke-timeout:The maximum timeout (in milliseconds) for asynchronous calls. The default value for this configuration is 5000.
> * refresh-endpoint-interval:The interval (in milliseconds) for periodically accessing the registry to obtain information. The default value for this configuration is one minute.
> * stat:The address of the service is called between modules. If this item is not configured, it means that the reported data will be directly discarded.
> * property:The address that the service reports its attribute. If it is not configured, this means that the reported data is directly discarded.
> * report-interval:The interval at which the information is reported to stat/property. The default is 60000 milliseconds.
> * asyncthread:The number of threads that process asynchronous responses when taking an asynchronous call. The default is 1.
> * modulename:The module name, the default value is the name of the executable program.

The format of the communicator's configuration file is as follows:
```
<tars>
  <application>
    #The configuration required by the proxy
    <client>
        #address
        locator                     = tars.tarsregistry.QueryObj@tcp -h 127.0.0.1 -p 17890
        #The maximum timeout (in milliseconds) for synchronous calls.
        sync-invoke-timeout         = 3000
        #The maximum timeout (in milliseconds) for asynchronous calls.
        async-invoke-timeout        = 5000
        #The maximum timeout (in milliseconds) for synchronous calls.
        refresh-endpoint-interval   = 60000
        #Used for inter-module calls
        stat                        = tars.tarsstat.StatObj
        #Address used for attribute reporting
        property                    = tars.tarsproperty.PropertyObj
        #report time interval
        report-interval             = 60000
        #The number of threads that process asynchronous responses
        asyncthread                 = 3
        #The module name
        modulename                  = Test.HelloServer
    </client>
  </application>
</tars>
```
Instructions for use:
> * When using the Tars framework for server use, users do not need to create their own communicators, directly using the communicator in the service framework. E.g: Application::getCommunicator()->stringToProxy(...). For a pure client scenario, the user needs to define a communicator and generate a service proxy.
> * Application::getCommunicator() is a static function of the Application class, which can be obtained at any time;
> * For the service proxy created by the communicator, it is also not necessary to call stringToProxy() before each use. The service proxy will be established during initialization, and it can be used directly afterwards.
> * For the creation and use of agents, please see the following sections;
> * For the same Obj name, the service proxy obtained by calling stringToProxy() multiple times is actually the same variable, which is safe for multi-threaded calls and does not affect performance.
> * The ip list corresponding to obj can be obtained by Application::getCommunicator()->getEndpoint("obj").Another way to get an IP list is to get it directly from the proxy generated by the communicator, such as Application::getCommunicator()->stringToProxy(...) ->getEndpoint().

## 2.2. Timeout control

The timeout control is for the client proxy. There are records in the configuration file of the communicator described in the previous section:
```cpp
#The maximum timeout (in milliseconds) for synchronous calls.
sync-invoke-timeout          = 3000
#The maximum timeout (in milliseconds) for asynchronous calls.
async-invoke-timeout         = 5000
```
The above timeout is valid for all the proxies generated by the communicator.

If you need to set the timeout separately, as shown below:

Set the timeout period for the proxy:
```cpp
ProxyPrx  pproxy;
//Set the timeout for the agent's synchronous call (in milliseconds)
pproxy->tars_timeout(3000);
//Sets the timeout for the agent's asynchronous call (in milliseconds)
pproxy->tars_async_timeout(4000);
```

Set the timeout for the calling interface:
```cpp
//Set the timeout (in milliseconds) for this interface call of this agent. This setting will only take effect once.
pproxy->tars_set_timeout(2000)->a(); 
```

## 2.3. Call interface

This section details how the Tars client remotely invokes the server.

First, briefly describe the addressing mode of the Tars client. Secondly, it will introduce the calling method of the client, including but not limited to one-way calling, synchronous calling, asynchronous calling, hash calling, and so on.

### 2.3.1. Introduction to addressing mode

The addressing mode of the Tars service can usually be divided into two ways: the service name is registered in the master and the service name is not registered in the master. A master is a name server (routing server) dedicated to registering service node information.

The service name added in the name server is implemented through the operation management platform.

For services that are not registered with the master, it can be classified as direct addressing, that is, the ip address of the service provider needs to be specified before calling the service. The client needs to specify the specific address of the HelloObj object when calling the service:

that is: Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985

Test.HelloServer.HelloObj: ¶ÔÏóÃû³Æ

tcp:Tcp protocol

-h:Specify the host address, here is 127.0.0.1

-p:Port, here is 9985

If HelloServer is running on two servers, HelloPrx is initialized as follows:
```cpp
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985:tcp -h 192.168.1.1 -p 9983");
```
The address of HelloObj is set to the address of the two servers. At this point, the request will be distributed to two servers (distribution method can be specified, not introduced here). If one server is down, the request will be automatically assigned to another one, and the server will be restarted periodically.

For services registered in the master, the service is addressed based on the service name. When the client requests the service, it does not need to specify the specific address of the HelloServer, but it needs to specify the address of the `registry` when generating the communicator or initializing the communicator.

The following shows the address of the registry by setting the parameters of the communicator:
```
CommunicatorPtr c = new Communicator();
c->setProperty("locator", "tars.tarsregistry.QueryObj@tcp -h .. -p ..")
```
Since the client needs to rely on the registry's address, the registry must also be fault-tolerant. The registry's fault-tolerant method is the same as above, specifying the address of the two registry.

### 2.3.2. One-way call

A one-way call means that the client only sends data to the server without receiving the response from the server, and whether the server receives the request data.
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//Initialize the communicator with a configuration file
c-> setProperty(conf);
//Generate a client's service proxy
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985");
//Initiate a remote call
string s = "hello word";
string r;
pPrx->async_testHello(NULL, s);
```
### 2.3.3. Synchronous call

Take a look at the code example below:
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//Initialize the communicator with a configuration file
c-> setProperty(conf);
//Generate a client's service proxy
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985");
//Initiate a remote synchronization call
string s = "hello word";
string r;
int ret = pPrx->testHello(s, r);
assert(ret == 0);
assert(s == r);
```
The above example shows that the client initiates a remote synchronization call to the HelloObj object of the HelloServer.

### 2.3.4. Asynchronous call

Define an asynchronous callback object:
```cpp
struct HelloCallback : public HelloPrxCallback
{
//Callback
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
//Initialize the communicator with a configuration file
c-> setProperty(conf);
//Generate a client's service proxy
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985");
//Define an object for a remote callback class
HelloPrxCallbackPtr cb = new HelloCallback;

//Initiate a remote synchronization call
string s = "hello word";
string r;
pPrx->async_testHello(cb, s);
```
note:
> * When a response from the server is received, HelloPrxCallback::callback_testHello() will be called.
> * If the asynchronous call returns an exception or timeout, then HelloPrxCallback::callback_testHello_exception() will be called with the return value defined as follows:


```cpp
//The return code given by the TARS server
const int TARSSERVERSUCCESS       = 0;       //The server is successfully processed
const int TARSSERVERDECODEERR     = -1;      //Server decoding exception
const int TARSSERVERENCODEERR     = -2;      //Server encoding exception
const int TARSSERVERNOFUNCERR     = -3;      //The server does not have this function
const int TARSSERVERNOSERVANTERR  = -4;      //The server does not have the Servant object.
const int TARSSERVERRESETGRID     = -5;      //Inconsistent gray state on the server side
const int TARSSERVERQUEUETIMEOUT  = -6;      //Server queue exceeded limit
const int TARSASYNCCALLTIMEOUT    = -7;      //Asynchronous call timeout
const int TARSINVOKETIMEOUT       = -7;      //Call timeout
const int TARSPROXYCONNECTERR     = -8;      //Proxy link exception
const int TARSSERVEROVERLOAD      = -9;      //The server is overloaded and exceeds the queue length.
const int TARSADAPTERNULL         = -10;     //The client routing is empty, the service does not exist or all services are offline.
const int TARSINVOKEBYINVALIDESET = -11;     //The client is called by an invalid set rule
const int TARSCLIENTDECODEERR     = -12;     //Client decoding exception
const int TARSSERVERUNKNOWNERR    = -99;     //Server location is abnormal
```

### 2.3.5. Set mode call

Currently, the framework already supports the deployment of services in set mode. After deployment by set, calls between services are transparent to business development. However, because some services have special requirements, after the deployment by set, the client can specify the set name to invoke the server. So the framework adds the ability for the client to specify the set name to call those services deployed by set.

The detailed usage rules are as follows:

Assume that the service server HelloServer is deployed on two sets, Test.s.1 and Test.n.1. Then the client specifies the set mode to be called as follows:
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//Initialize the communicator with a configuration file
c-> setProperty(conf);
//Generate a client's service proxy
HelloPrx pPrx_Tests1 = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985","Test.s.1");

HelloPrx pPrx_Testn1 = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985","Test.n.1");

//Initiate a remote synchronization call
string s = "hello word";
string r;

int ret = pPrx_Tests1->testHello(s, r);

int ret = pPrx_Testn1->testHello(s, r);
```
note:
> * The priority of the specified set call is higher than the priority of the client and the server itself to enable the set. For example, both the client and the server have "Test.s.1" enabled, and if the client specifies "Test.n.1" when creating the server proxy instance, the actual request is sent to "Test.n.1"("Test.n.1" has a deployment service).
> * Just create a proxy instance once

### 2.3.6. Hash call

Since multiple servers can be deployed, client requests are randomly distributed to the server, but in some cases, it is desirable that certain requests are always sent to a particular server. In this case, Tars provides a simple way to achieve:

If there is a request for querying data according to the QQ number, as follows:
```cpp
QQInfo qi = pPrx->query(uin);
```
Normally, for the same call to uin, the server address of each response is not necessarily the same. However, With the following call, it can be guaranteed that each request for uin is the same server response.
```cpp
QQInfo qi = pPrx->tars_hash(uin)->query(uin);
```


note:
> * This method is not strict. If a server goes down, these requests will be migrated to other servers. When it is normal, the request will be migrated back.
> * The argument to tars_hash() must be int. For string, the Tars base library (under the util directory) also provides the method: tars::hash<string>()("abc"). See util/tc_hash_fun.h for details.