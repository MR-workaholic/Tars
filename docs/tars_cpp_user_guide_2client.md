# 2. C++�ͻ���

�ͻ��˿��Բ���д�κ�Э��ͨ�Ŵ��뼴�����Զ�̵��á��ͻ��˴���ͬ����Ҫ����hello.h�ļ���
## 2.1. ͨ����

��ɷ�����Ժ󣬿ͻ�����Ҫ�Է��������շ����Ĺ������ͻ��˶Է��������շ����Ĳ�����ͨ��ͨ������communicator����ʵ�ֵġ�

** ע��:һ��Tars����ֻ��ʹ��һ��Communicator����Application::getCommunicator()��ȡ���ɣ�������Tars����Ҫʹ��ͨѶ�������Լ�������Ҳֻ��һ������

ͨ�����ǿͻ��˵���Դ���壬������һ����Դ�������շ�����״̬ͳ�Ƶȹ��ܡ�

ͨ�����ĳ�ʼ�����£�
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//�������ļ���ʼ��ͨ����
c-> setProperty(conf);
//ֱ�������Գ�ʼ��
c->setProperty("property", "tars.tarsproperty.PropertyObj");
c->setProperty("locator", "tars.tarsregistry.QueryObj@tcp -h ... -p ...");
```
˵����
> * ͨ�����������ļ���ʽ��������ܣ�
> * ͨ����ȱʡ�����������ļ�Ҳ����ʹ�ã����в�������Ĭ��ֵ��
> * ͨ����Ҳ����ֱ��ͨ����������ɳ�ʼ����
> * �����Ҫͨ����������ȡ�ͻ��˵��ô������������locator������

ͨ��������˵����
> * locator: registry����ĵ�ַ����������ip port�ģ��������Ҫregistry����λ��������Ҫ���ã�
> * sync-invoke-timeout���������ʱʱ�䣨ͬ���������룬û������ȱʡΪ3000
> * async-invoke-timeout���������ʱʱ�䣨�첽�������룬û������ȱʡΪ5000
> * refresh-endpoint-interval����ʱȥregistryˢ�����õ�ʱ���������룬û������ȱʡΪ1����
> * stat��ģ�����÷���ĵ�ַ�����û�����ã����ϱ�������ֱ�Ӷ�����
> * property�������ϱ���ַ�����û�����ã����ϱ�������ֱ�Ӷ�����
> * report-interval���ϱ���stat/property��ʱ������Ĭ��Ϊ60000���룻
> * asyncthread���첽����ʱ���ص��̵߳ĸ�����Ĭ��Ϊ1��
> * modulename��ģ�����ƣ�Ĭ��Ϊ��ִ�г������ƣ�

ͨ���������ļ���ʽ���£�
```
<tars>
  <application>
    #proxy��Ҫ������
    <client>
        #��ַ
        locator                     = tars.tarsregistry.QueryObj@tcp -h 127.0.0.1 -p 17890
        #ͬ�����ʱʱ��(����)
        sync-invoke-timeout         = 3000
        #�첽���ʱʱ��(����)
        async-invoke-timeout        = 5000
        #ˢ�¶˿�ʱ����(����)
        refresh-endpoint-interval   = 60000
        #ģ������
        stat                        = tars.tarsstat.StatObj
        #�����ϱ���ַ
        property                    = tars.tarsproperty.PropertyObj
        #report time interval
        report-interval             = 60000
        #�����첽�ص��̸߳���
        asyncthread                 = 3
        #ģ������
        modulename                  = Test.HelloServer
    </client>
  </application>
</tars>
```
ʹ��˵����
> * ��ʹ��Tars����������ʹ��ʱ��ͨ������Ҫ�Լ�������ֱ�Ӳ��÷������е�ͨ�����Ϳ����ˣ����磺Application::getCommunicator()->stringToProxy(...)�����ڴ��ͻ������Σ���Ҫ�û��Լ������ͨ���������ɴ���proxy����
> * getCommunicator()�ǿ��Application��ľ�̬��������ʱ���Ի�ȡ��
> * ����ͨ�������������Ĵ���Ҳ����Ҫÿ��ʹ�õ�ʱ��stringToProxy����ʼ��ʱ�����ã�����ֱ��ʹ�þͿ����ˣ�
> * ����Ĵ�����ʹ�ã���μ����漸�ڣ�
> * ��ͬһ��Obj���ƣ���ε���stringToProxy���ص�Proxy��ʵ��һ�������̵߳����ǰ�ȫ�ģ��Ҳ���Ӱ�����ܣ�
> * ����ͨ��Application::getCommunicator()->getEndpoint����obj������ȡobj��Ӧip�б�����һ�ֻ�ȡ��ʽΪֱ�Ӵ�ͨ�������ɵ�proxy��ȡ��Application::getCommunicator()->stringToProxy(��) ->getEndpoint()

## 2.2. ��ʱ����

��ʱ�����ǶԿͻ���proxy���������ԡ��Ͻ���ͨ�����������ļ����м�¼��
```cpp
#ͬ�����ʱʱ��(����)
sync-invoke-timeout          = 3000
#�첽���ʱʱ��(����)
async-invoke-timeout         = 5000
```
����ĳ�ʱʱ���ͨ�������ɵ�����proxy����Ч��

�����Ҫ�������ó�ʱʱ�䣬�������£�

���proxy���ó�ʱʱ�䣺
```cpp
ProxyPrx  pproxy;
//���øô����ͬ���ĵ��ó�ʱʱ�� ��λ����
pproxy->tars_timeout(3000);
//���øô�����첽�ĵ��ó�ʱʱ�� ��λ����
pproxy->tars_async_timeout(4000);
```

��Խӿ����ó�ʱʱ�䣺
```cpp
//���øô���ı��νӿڵ��õĳ�ʱʱ��.��λ�Ǻ��룬���õ�����Ч
pproxy->tars_set_timeout(2000)->a(); 
```

## 2.3. ����

���ڻ���ϸ����tars��Զ�̵��õķ�ʽ��

���ȼ���tars�ͻ��˵�Ѱַ��ʽ����λ���ܿͻ��˵ĵ��÷�ʽ�������������ڵ�����á�ͬ�����á��첽���á�hash���õȡ�

### 2.3.1. Ѱַ��ʽ���

Tars�����Ѱַ��ʽͨ�����Է�Ϊ�������ַ�ʽ����������������ע��Ͳ�������ע�ᣬ������ָר����ע�����ڵ���Ϣ�����ַ���·�ɷ��񣩡�
���ַ����еķ������������ͨ����������ƽ̨ʵ�ֵġ�

����û��������ע��ķ��񣬿��Թ�Ϊֱ��Ѱַ��ʽ�����ڷ����obj����ָ��Ҫ���ʵ�ip��ַ���ͻ����ڵ��õ�ʱ����Ҫָ��HelloObj����ľ����ַ��

����Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985

Test.HelloServer.HelloObj����������

tcp��tcpЭ��

-h��ָ��������ַ��������127.0.0.1 

-p���˿ڵ�ַ��������9985

���HelloServer����̨�����������У���HelloPrx��ʼ����ʽ���£�
```cpp
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985:tcp -h 192.168.1.1 -p 9983");
```
����HelloObj�ĵ�ַ����Ϊ��̨�������ĵ�ַ����ʱ�����ַ�����̨�������ϣ��ַ���ʽ����ָ�������ﲻ�����ܣ������һ̨������down�����Զ�������ֵ�����һ̨������ʱ���Կ�ʼdown����һ̨��������

������������ע��ķ��񣬷����Ѱַ��ʽ�ǻ��ڷ��������еģ��ͻ������������˵�ʱ������Ҫָ��HelloServer�ľ����ַ��������Ҫ������ͨ�������ʼ��ͨ������ʱ��ָ��registry(��������)�ĵ�ַ��

����ָ��registry�ĵ�ַͨ������ͨ�����Ĳ���ʵ�֣����£�
```
CommunicatorPtr c = new Communicator();
c->setProperty("locator", "tars.tarsregistry.QueryObj@tcp -h .. -p ..")
```
��������registry�ĵ�ַ�����registry����Ҳ�ܹ��ݴ�����registry���ݴ�ʽ��������һ����ָ������registry�ĵ�ַ���ɡ�

### 2.3.2. �������

��ν������ã���ʾ�ͻ���ֻ�ܷ������ݣ��������շ���˵���Ӧ��Ҳ���ܷ�����Ƿ���յ�����
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//�������ļ���ʼ��ͨ����
c-> setProperty(conf);
//���ɿͻ��˴���
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985");
//����Զ�̵���
string s = "hello word";
string r;
pPrx->async_testHello(NULL, s);
```
### 2.3.3. ͬ������

�뿴���µ���ʾ����
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//�������ļ���ʼ��ͨ����
c-> setProperty(conf);
//���ɿͻ��˴���
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985");
//����Զ��ͬ������
string s = "hello word";
string r;
int ret = pPrx->testHello(s, r);
assert(ret == 0);
assert(s == r);
```
���������б�ʾ���ͻ�����HelloServer��HelloObj������Զ��ͬ�����á�

### 2.3.4. �첽����

�����첽�ص�����
```cpp
struct HelloCallback : public HelloPrxCallback
{
//�ص�����
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
//�������ļ���ʼ��ͨ����
c-> setProperty(conf);
//���ɿͻ��˴���
HelloPrx pPrx = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985");
//����Զ�̻ص�����
HelloPrxCallbackPtr cb = new HelloCallback;

//����Զ�̵���
string s = "hello word";
string r;
pPrx->async_testHello(cb, s);
```
ע�⣺
> * �����յ�����˷���ʱ��HelloPrxCallback��callback_testHello�ᱻ��Ӧ��
> * ������÷����쳣��ʱ����callback_testHello_exception�ᱻ���ã�ret��ֵ�������£�


```cpp
//����TARS��������ķ�����
const int TARSSERVERSUCCESS       = 0;       //�������˴���ɹ�
const int TARSSERVERDECODEERR     = -1;      //�������˽����쳣
const int TARSSERVERENCODEERR     = -2;      //�������˱����쳣
const int TARSSERVERNOFUNCERR     = -3;      //��������û�иú���
const int TARSSERVERNOSERVANTERR  = -4;      //��������û�и�Servant����
const int TARSSERVERRESETGRID     = -5;      //�������˻Ҷ�״̬��һ��
const int TARSSERVERQUEUETIMEOUT  = -6;      //���������г�������
const int TARSASYNCCALLTIMEOUT    = -7;      //�첽���ó�ʱ
const int TARSINVOKETIMEOUT       = -7;      //���ó�ʱ
const int TARSPROXYCONNECTERR     = -8;      //proxy�����쳣
const int TARSSERVEROVERLOAD      = -9;      //�������˳�����,�������г���
const int TARSADAPTERNULL         = -10;     //�ͻ���ѡ·Ϊ�գ����񲻴��ڻ������з���down����
const int TARSINVOKEBYINVALIDESET = -11;     //�ͻ��˰�set������÷Ƿ�
const int TARSCLIENTDECODEERR     = -12;     //�ͻ��˽����쳣
const int TARSSERVERUNKNOWNERR    = -99;     //��������λ���쳣
```

### 2.3.5. ָ��set��ʽ����

Ŀǰ����Ѿ�֧��ҵ��set��ʽ���в��𣬰�set����֮�󣬸���ҵ��֮��ĵ��ö�ҵ�񿪷���˵��͸���ġ�����������Щҵ��������������Ҫ�ڰ�set����֮�󣬿ͻ��˿���ָ��set���������÷���ˣ���˿����set����Ļ����������˿ͻ��˿���ָ��set����ȥ����ҵ�����Ĺ��ܡ�

��ϸʹ�ù������£�

����ҵ������HelloServer����������set�ϣ��ֱ�ΪTest.s.1��Test.n.1����ô�ͻ���ָ��set��ʽ�������£�
```cpp
TC_Config conf("config.conf");
CommunicatorPtr c = new Communicator();
//�������ļ���ʼ��ͨ����
c-> setProperty(conf);
//���ɿͻ��˴���
HelloPrx pPrx_Tests1 = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985","Test.s.1");

HelloPrx pPrx_Testn1 = c->stringToProxy<HelloPrx>("Test.HelloServer.HelloObj@tcp -h 127.0.0.1 -p 9985","Test.n.1");

//����Զ�̵���
string s = "hello word";
string r;

int ret = pPrx_Tests1->testHello(s, r);

int ret = pPrx_Testn1->testHello(s, r);
```
ע�⣺
> * ָ��set���õ����ȼ����ڿͻ��˺ͷ������������set�����ȼ������磬�ͻ��˺ͷ���˶�������Test.s.1������ͻ��˴�������˴���ʵ��ʱָ����Test.n.1����ôʵ������ͻᷢ�͵�Test.n.1�Ľڵ��ϣ�ǰ�������set�в������ˣ���
> * �����Ĵ���ʵ��Ҳ��ֻ�����һ��

### 2.3.6. Hash����

���ڷ�����Բ����̨������Ҳ������ַ��������ϣ�������ĳЩ�����£�ϣ��ĳЩ����������ĳһ̨�������ϣ������������Tars�ṩ�˼򵥵ķ�ʽʵ�֣�

������һ������QQ�Ų�ѯ���ϵ��������£�
```cpp
QQInfo qi = pPrx->query(uin);
```
ͨ������£�ͬһ��uin���ã�ÿ���������ڵķ�������ַ����һ�������ǲ������µ��ã�
```cpp
QQInfo qi = pPrx->tars_hash(uin)->query(uin);
```
����Ա�֤��ÿ��uin��������ĳһ̨��������

ע�⣺
> * ���ַ�ʽ�����ϸ������̨ĳ̨������down������Щ�����Ǩ�Ƶ����������������������󣬻�Ǩ�ƻ�����
> * Hash����������int������string��˵��Tars������(utilĿ¼��)Ҳ�ṩ�˷�����tars::hash<string>()("abc")��������μ�util/tc_hash_fun.h