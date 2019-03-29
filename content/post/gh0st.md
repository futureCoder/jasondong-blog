`CIniFile`读取配置文件, `CGh0stApp`在其重载的`InitInstance()`中, 调用主窗口`CMainFrame`的`Activate(UINT nPort, UINT nMaxConnections)`函数并将`CIniFile`中的配置传入.  
`CMainFrame::Activate`主要只做了一件事, 就是创建一个`CIOCPServer`的实例并且调用`CIOCPServer::Initialize(NOTIFYPROC pNotifyProc, CMainFrame* pFrame,  int nMaxConnections, int nPort)`进行初始化, 其中, **参数1**`pNotifyProc`是一个回调函数, 传入值是`CMainFrame::NotifyProc(LPVOID lpParam, ClientContext* pContext, UINT nCode)`函数,其作用是将一条`WM_UPDATE_MAINFRAME`消息放入消息队列并附加两个消息参数nCode和pContent.**参数2**是主窗口指针,**参数3**是最大连接数,**参数4**是监听端口.`Initialize`的前三个参数全被`CIOCPServer`存至成员变量,下面来介绍`CIOCPServer::Initialize`的逻辑.
1. 创建异步I/O套接字m_socListen; 搜一下引用可以看到m_socListen被ListenThreadProc、OnAccept和Shutdown三个函数所使用;
2. WSACreateEvent()创建事件对象m_hEvent,并使用WSAEventSelect事件选择模型,只关心FD_ACCEPT事件; m_hEvent被ListenThreadProc和Shutdown使用;
3. bind, listen没什么好说的, 之后创建并启动一个监听线程`ListenThreadProc`;
4. 此时，初始化还没完，监听线程创建成功后, `CIOCPServer::Initialize`会继续执行`InitializeIOCP`函数;
5. `InitializeIOCP`创建一个socket并根据这个socket创建一个IO完成端口m_hCompletionPort,之后启动了(2*CPU核心数)个ThreadPoolFunc线程; m_hCompletionPort 被ThreadPoolFunc、OnAccept、Send和CloseCompletionPort使用.
6. `CIOCPServer::Initialize`运行结束

可以看到`CIOCPServer::Initialize`主要做了三件事:
* 创建异步I/O套接字m_socListen, 创建事件对象m_hEvent, 创建IO完成端口m_hCompletionPort;
* 启动一个监听线程`ListenThreadProc`
* 启动若干个`ThreadPoolFunc`
同时,知道了除去Shutdown和close一类的函数, ListenThreadProc需要m_socListen和m_hEvent数据，OnAccept需要m_socListen和m_hCompletionPort，ThreadPoolFunc需要m_hCompletionPort数据，Send需要m_hCompletionPort。

接下来看一下线程ListenThreadProc都做了些什么。

```
while(1)
{
1    if (WaitForSingleObject(pThis->m_hKillEvent, 100) == WAIT_OBJECT_0)     //若在100ms内等到了Stop()函数设置的m_hKillEvent事件,则线程退出,Stop等待此线程退出后继续进行清理操作. **这里不懂为什么要用event而不是成员变量来控制while的条件.**
2            break;
3    WSAWaitForMultipleEvents(1, &pThis->m_hEvent, FALSE, 100, FALSE);       //若在100ms内等pThis->m_hEvent的任何信号, 根据是否有信号来选择下一步操作.
4    WSAEnumNetworkEvents(pThis->m_socListen, pThis->m_hEvent, &events);     //为m_socListen查看已经发生的网络事件, 放到events里, 并清掉pThis->m_hEvent的事件.
5    if (events.lNetworkEvents & FD_ACCEPT)                                  //如果有客户端连接事件发生
6            if (events.iErrorCode[FD_ACCEPT_BIT] == 0)                      //并且对应位没有报告错误
7                pThis->OnAccept();                                          //执行接受连接的逻辑.
}
```
ListenThreadProc的逻辑很简单, 先看程序是否退出了,退出则线程返回(1-2), 再看是否有客户端连接事件发生(3-6), 最后接受连接(7).

由于OnAccept是处理客户端的第一步操作, 并且访问了m_socListen和m_hCompletionPort, 而ThreadPoolFunc又需要m_hCompletionPort数据, 推测两者存在生产消费之类的关系, 所以还是先从OnAccept看起. 
OnAccept的主要逻辑:
{

    	clientSocket = accept(m_socListen, (LPSOCKADDR)&SockAddr, &nLen);   //接受客户端连接
    	ClientContext* pContext = AllocateContext();    //创建一个抽象出来的I/O完成端口需要的读写结构.
        pContext->m_Socket = clientSocket;      //pContext持有客户端连接
        pContext->m_wsaInBuffer.buf = (char*)pContext->m_byInBuffer;        //BYTE[8192]给WSABUF
        pContext->m_wsaInBuffer.len = sizeof(pContext->m_byInBuffer);       
    
        CreateIoCompletionPort((HANDLE)clientSocket, m_hCompletionPort, (DWORD)pContext, 0);    //将clientSocket绑定到m_hCompletionPort上, 并设置pContext为userData.
    
        //对pContext->m_Socket开启保活机制并设置保活信息.
    
        CLock cs(m_cs, "OnAccept");
        m_listContexts.AddTail(pContext);   //加入m_listContexts统一管理
    
        OVERLAPPEDPLUS	*pOverlap = new OVERLAPPEDPLUS(IOInitialize);               //没看懂OVERLAPPED的作用和把OVERLAPPED封了一层干什么用的, 只是为了自解释么...---看了ThreadPoolFunc知道了把OVERLAPPED封了一层是标识要对pContext进行的操作
        PostQueuedCompletionStatus(m_hCompletionPort, 0, (DWORD)pContext, &pOverlap->m_ol);     //只查到这个函数意义是发送一个完成包到完成端口, 似乎可以唤醒执行了GetQueuedCompletionStatus等待m_hCompletionPort消息的线程, 时间关系具体过程和原理暂时没查清楚.
    
        m_pFrame->PostMessage(WM_WORKTHREAD_MSG, NC_CLIENT_CONNECT, (LPARAM)pContext);  //
    
        OVERLAPPEDPLUS * pOverlap = new OVERLAPPEDPLUS(IORead);
        ULONG			ulFlags = MSG_PARTIAL;
        DWORD			dwNumberOfBytesRecvd;
        WSARecv(pContext->m_Socket, &pContext->m_wsaInBuffer, 1, &dwNumberOfBytesRecvd, &ulFlags, &pOverlap->m_ol, NULL);       //提供pContext->m_wsaInBuffer作为接收pContext->m_Socket发来数据的缓冲区
}
总结一下, accetp客户端的连接并拿到cltsock, 对cltsock设置保活, 创建一个"用来保存cltsock和持有收发数据缓冲区"等作用的抽象结构ClientContext, 随后创建IO完成端口并绑定cltsock, 通过PostQueuedCompletionStatus唤醒等待在GetQueuedCompletionStatus上的线程并把ClientContext的地址给他好让他做点什么(猜的), 最后设置用于接收cltsock数据的缓冲区,好让操作系统帮我们自动接收数据(并自动通过完成端口通知我们么??).

再看一下另外一个线程ThreadPoolFunc的逻辑.



`#define CONTAINING_RECORD(memberAddress, classType, memmer) ( (classType *) ( (PCHAR) (memberAddress) - (ULONG_PTR) (& (((classType *)0)->memmer) ) ) )`
用类成员变量的地址减去成员变量在类内的地址偏移, 得到类地址.  

