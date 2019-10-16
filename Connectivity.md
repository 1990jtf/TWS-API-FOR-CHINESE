# 连接性
API客户端应用程序与TWS之间的套接字连接是通过IBApi.EClientSocket.eConnect函数建立的。TWS充当服务器来接收来自API应用程序（客户端）的请求，并通过采取适当的措施做出响应。第一步是让API客户端在已经监听TWS的套接字端口上启动与TWS的连接。如果每个TWS实例配置了不同的API套接字端口号，则可能在同一台计算机上运行多个TWS实例。此外，每个TWS会话最多可以同时接收32个不同的客户端应用程序。API连接中指定的客户端ID字段用于区分不同的API客户端。

## 建立API连接
一旦创建了两个主要对象EWrapper和ESocketClient，客户端应用程序便可以通过**IBApi.EClientSocket**对象进行连接：  
```c++
bool bRes = m_pClient-> eConnect（host，port，clientId，m_extraAuth）;
```
eConnect首先从操作系统请求打开TCP套接字到指定的IP地址和套接字端口。如果无法打开套接字，则操作系统（非TWS）会将API客户端收到的错误作为错误代码502返回给IBApi.EWrapper.error（注意：由于该错误不是由TWS生成的，因此不会捕获到TWS日志文件）。最常见的错误502将指示TWS未在启用API的情况下运行，或者正在侦听其他套接字端口上的连接。如果通过网络进行连接，则在防火墙或防病毒程序阻止连接的情况下，或者在TWS的“受信任IP”中未列出路由器的IP地址的情况下，也会发生此错误。

打开套接字后，必须进行初始握手，在该握手中交换有关TWS和API支持的最高版本的信息。这很重要，因为API消息在不同版本中可以具有不同的长度和字段，并且必须具有版本号才能正确解释收到的消息。

+ 因此，重要的是在建立连接后才创建主EReader对象。初始连接会导致TWS与API客户端之间协商通用版本，而EReader线程在解释后续消息时将需要该通用版本。
建立可用于通信的最高版本号后，TWS将返回特定于登录的TWS用户会话的某些数据。这包括（1）在该TWS会话中可访问的帐号，（2）下一个有效的订单标识符（ID），以及（3）连接时间。在最常见的操作模式下，EClient.AsyncEConnect字段设置为false，并且在建立套接字连接后立即完成初始握手。然后，TWS将立即向API客户端提供此信息。

+ 重要提示：IBApi.EWrapper.nextValidID回调通常用于指示连接已完成，并且其他消息可以从API客户端发送到TWS。TWS可能会放弃在此之前进行的函数调用。  

在特殊情况下，存在另一种不推荐使用的连接模式，其中变量AsyncEconnect设置为true，并且仅从connectAck（）函数调用startAPI。所有IB样本都使用模式AsyncEconnect = False。

## EReader 线程
API程序始终至少具有两个执行线程。一个线程用于将消息发送到TWS，另一个线程用于读取返回的消息。第二个线程使用API​​ EReader类从套接字读取并将消息添加到队列。每次将新消息添加到消息队列时，都会有一个通知标志被触发，以使其他线程现在可以等待处理一条消息。  

在API程序的双线程设计中，消息队列也由第一个线程处理。在三线程设计中，将创建一个附加线程来执行此任务。负责消息队列的线程将解码消息并在EWrapper中调用适当的功能。  

IB Python示例Program.py和C ++示例TestCppClient中使用了两线程设计，而“ Testbed” 其他语言的样本使用三线程设计。通常在Python异步网络应用程序中，asyncio模块将用于创建更具顺序感的代码设计。

具有读取和解析来自TWS的原始消息的功能的类是IBApi.EReader类。
```c++
m_pReader = new EReader（m_pClient，＆m_osSignal);
m_pReader -> start();
```

现在是时候重新审视的作用IBApi.EReaderSignal在最初推出的EClientSocket类。  
如上一段所述，在EReader线程将消息放入队列后，将发出通知以告知消息已准备好进行处理。  
在（C ++, C＃, .NET, Java）API中，这是通过我们在IBApi.EWrapper的实现程序中启动的IBApi.EReaderSignal对象完成的。  
在Python API中，它由Queue类自动处理。

客户端应用程序现在可以与交易者工作站一起使用了！连接完成后，API程序将开始接收事件，例如IBApi.EWrapper.nextValidId和IBApi.EWrapper.managedAccounts。在交易平台（没有IB网关），如果有一个活动的网络连接，也会有立即回调IBApi :: EWrapper ::错误与ErrorID中为-1，错误码= 2104，2106，ERRORMSG =“市场数据服务器就可以了”表示存在与IB市场数据服务器的有效连接。回调到IBApi :: EWrapper :: error errorId为-1的情况并不表示真正的“错误”，而仅表示已成功连接到IB市场数据场的通知。

相反，在IB客户端发出请求之前，IB网关不会建立与市场数据场的连接。在此之前，IB网关GUI中的连接指示器将显示黄色的“不活动”，而不是“活动”绿色指示。

最初从API应用程序发出请求时，重要的是要验证是否已收到响应，而不是假定网络连接正常并且已成功进行预订请求（资产更新，帐户信息等），然后继续进行。

## 接受来自TWS的API连接
出于安全原因，默认情况下，未将API配置为自动接受来自API应用程序的连接请求。尝试连接后，将在TWS中出现一个对话框，要求用户手动确认可以建立连接：  

![avatar](https://interactivebrokers.github.io/tws-api/conn_prompt.png)

为了防止TWS要求最终用户接受连接，可以将其配置为自动接受来自受信任IP地址和/或本地计算机的连接。可以通过TWS API设置轻松完成此操作： 

![avatar](https://interactivebrokers.github.io/tws-api/tws_allow_connections.png)

注意：在尝试向TWS发送任何请求之前，必须确保已完全建立连接。否则，将导致TWS关闭连接。通常，这可以通过等待事件的回调和初始连接握手（例如IBApi.EWrapper.nextValidId或IBApi.EWrapper.managedAccounts）的结束来完成。

在极少数情况下，IB网关或TWS在建立与IB服务器的连接时会短暂延迟，在接收nextValidId之后立即发送的消息可能会被丢弃并且需要重新发送。如果API客户端尚未从发出的请求中收到预期的回调，则假定连接正常，则不应继续进行。

## API套接字连接断开
如果TWS与API客户端之间的套接字连接出现问题，例如，如果TWS突然关闭，这将在EReader线程中触发一个异常，该线程正在从套接字读取。如果API客户端尝试使用已在使用的客户端ID连接，也会发生此异常。

套接字EOF在不同的API语言中的处理方式略有不同。  
例如，在Java中，它被捕获并发送到客户端应用程序到IBApi :: EWrapper :: error，错误代码为507：“错误消息”。在C＃中，它被捕获并发送到IBApi :: EWrapper :: error，错误代码为-1。客户端应用程序需要处理此错误消息，并使用它来指示套接字连接中已引发异常。API代码不会自动调用相关函数，例如IBApi :: EWrapper :: connectionClosed和IBApi :: EClient :: IsConnected函数，但需要在API客户端级别*进行处理。

+ API版本973.04中对此进行了更改