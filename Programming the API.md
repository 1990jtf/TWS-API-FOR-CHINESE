# 编程API：架构
## EClientSocket和EWrapper类
一旦TWS启动并运行并积极监听传入的连接，我们就可以编写代码了。这使我们进入了TWS API的两个主要类：IBApi.EWrapper接口和IBApi.EClientSocket
## 实现EWrapper接口
所述IBApi.EWrapper接口是通过该输送TWS信息给API客户端应用程序的机制。通过实现此接口，客户端应用程序将能够接收和处理来自TWS的信息。有关如何实现接口的更多信息，请参考编程语言的文档。
```c++
class TestCppClient：public EWrapper
{
    
}
```
## EClientSocket类
用于向TWS发送消息的类是IBApi.EClientSocket。与EWrapper不同，此类不会被覆盖，因为将调用EClientSocket中提供的函数将消息发送到TWS。  
要使用EClientSocket，首先可能需要实现IBApi.EWrapper接口作为其构造函数参数的一部分，以便应用程序可以处理所有返回的消息。从TWS发送的消息作为对IBApi.EClientSocket中函数调用的响应，需要EWrapper实现，以便可以对其进行处理以满足API客户端的需求。

另一个关键元素是传递给EClientSocket构造函数的IBApi.EReaderSignal对象。除Python之外，此对象在API中使用，以表示已准备好在队列中进行处理的消息。（在Python中，Queue类直接处理此任务）。我们将在“电子阅读器线程”部分中更详细地讨论此对象。
```c++
EReaderOSSignal m_osSignal;
EClientSocket * const m_pClient;
```
```c++
TestCppClient::TestCppClient() :
      m_osSignal(2000) // 2-seconds timeout
    , m_pClient(new EClientSocket(this, &m_osSignal))
    , m_state(ST_CONNECT)
    , m_sleepDeadline(0)
    , m_orderId(0)
    , m_pReader(0)
    , m_extraAuth(false)
{
}
```