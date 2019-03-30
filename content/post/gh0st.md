Server端:
`CIniFile`读取配置文件, `CGh0stApp`在其重载的`InitInstance()`中, 调用主窗口`CMainFrame`的`Activate(UINT nPort, UINT nMaxConnections)`函数并将`CIniFile`中的配置传入.  
```
CMainFrame::Activate(UINT nPort, UINT nMaxConnections)  //nPort-端口, nMaxConnections-最大连接数
{
    ...
    m_iocpServer = new CIOCPServer;     //创建一个CIOCPServer的实例
    m_iocpServer->Initialize(NotifyProc, this, 100000, nPort);  //传入监听端口和回调函数, 进行启服初始化
    ...
}
```
可以看到`CMainFrame::Activate`主要只做了一件事, 就是创建并初始化`m_iocpServer`, `Initialize`函数的**参数2**是主窗口指针,**参数3**是最大连接数,**参数4**是监听端口, **参数1**`pNotifyProc`是一个回调函数,函数如下:
```
CMainFrame::NotifyProc(LPVOID lpParam, ClientContext *pContext, UINT nCode) //lpParam-主窗口指针, pContext-客户端连接数据, nCode-事件标识
{
    if (pContext == NULL || pContext->m_DeCompressionBuffer.GetBufferLen() <= 0)
        return;
    CMainFrame* pFrame = (CMainFrame*)lpParam;
    pFrame->PostMessage(WM_UPDATE_MAINFRAME, (WPARAM)nCode, (LPARAM)pContext);
}
```
其作用过滤无效数据并转发消息`WM_UPDATE_MAINFRAME`, `CMainFrame::NotifyProc2`会处理此消息:
```
CMainFrame::NotifyProc2(WPARAM wParam, LPARAM lParam)   //wParam-事件标识, lParam-ClientContext*
{
    ClientContext* pContext = (ClientContext *)lParam;
    UINT nCode = (UINT)wParam;
    switch (nCode)
    {
        case NC_CLIENT_DISCONNECT:
            g_pConnectView->PostMessage(WM_REMOVEFROMLIST, 0, (LPARAM)pContext);    //疑问: 这样做的目的是什么?避免什么问题?
            break;
        case NC_RECEIVE:
            ProcessReceive(pContext);
            break;
        case NC_RECEIVE_COMPLETE:
            ProcessReceiveComplete(pContext);
    }
}
```
可以看到`CMainFrame::NotifyProc2`实际是在更新UI, 而且之前的结构应该是`CMainFrame::NotifyProc`直接更新UI, 之后改成了`CMainFrame::NotifyProc`发重绘UI的消息, 由CMainFrame::NotifyProc2异步处理. 这样做是为什么呢? `CMainFrame::NotifyProc`作为回调函数被`CIOCPServer`作为成员变量`m_pNotifyProc`持有, 被工作线程引用, 在`CMainFrame::NotifyProc`同步更新的话, 不仅效率较低, 并且频繁更新UI的话, UI假死会直接阻塞住调用线程. 所以被改为线程只发更新UI的消息并在消息参数中附带上相关的数据nCode和pContent, 由主线程处理处理更新UI的具体逻辑(没深入看, 感觉是这样, 但这里有个疑问, 对于`NC_CLIENT_DISCONNECT`消息的处理是非同步的, 这么做的目的是什么呢?)

上面介绍的是CIOCPServer::Initialize中回调函数的逻辑, 下面来看一下Initialize本身的逻辑:
```
bool CIOCPServer::Initialize(NOTIFYPROC pNotifyProc, CMainFrame* pFrame, int nMaxConnections, int nPort)
{
    m_socListen = WSASocket(AF_INET, SOCK_STREAM, 0, NULL, 0, WSA_FLAG_OVERLAPPED); //创建异步I/O套接字m_socListen; 搜一下引用可以看到m_socListen被ListenThreadProc、OnAccept和Shutdown三个函数所使用;
    m_hEvent = WSACreateEvent();    //创建一个事件对象, m_hEvent被ListenThreadProc和Shutdown使用;
    WSAEventSelect(m_socListen, m_hEvent, FD_ACCEPT);   //使用WSAEventSelect事件选择模型,只关心FD_ACCEPT事件
    bind(), listen();
    m_hThread =(HANDLE)_beginthreadex(NULL,	0, ListenThreadProc, (void*) this, 0, &dwThreadId); //创建监听线程ListenThreadProc
    if (m_hThread != INVALID_HANDLE_VALUE)      //如果监听线程创建成功
    {
        CIOCPServer::InitializeIOCP()
        {
            s = socket(AF_INET, SOCK_STREAM, IPPROTO_IP);   //创建一个socket
            m_hCompletionPort = CreateIoCompletionPort((HANDLE)s, NULL, 0, 0);  //创建IO完成端口并绑定s
            closesocket(s);     //归还内核对象
            for (i = 0; i < nWorkerCnt; i++)
            {
                hWorker = (HANDLE)_beginthreadex(NULL, 0, ThreadPoolFunc, (void*) this, 0, &nThreadID); //创建ThreadPoolFunc线程
                CloseHandle(hWorker);   //归还内核对象
                //调查得出: 线程句柄是一个内核对象, 线程是进程中的一个最小运行单位(由操作系统调度), 进程可以通过线程句柄操作线程, 若进程不需要对线程进行任何操作, 则可以对线程句柄进行close操作, 此举并不会关掉或影响线程, 只是表示不再使用该内核对象, 操作系统可以回收他用了.
                //个人猜测, 上面m_hCompletionPort绑定的socket也是同样的逻辑, socket被绑定到了完成端口m_hCompletionPort上, 后续不再需要直接通过socket句柄操作套接字了, 也就可以归还内核对象了. 
            }
        }
    }
}
```
总结一下:
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
```
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
```
总结一下, accetp客户端的连接并拿到cltsock, 对cltsock设置保活, 创建一个"用来保存cltsock和持有收发数据缓冲区"等作用的抽象结构ClientContext, 随后创建IO完成端口并绑定cltsock, 通过PostQueuedCompletionStatus唤醒等待在GetQueuedCompletionStatus上的线程并把ClientContext的地址给他好让他做点什么(猜的), 最后设置用于接收cltsock数据的缓冲区,好让操作系统帮我们自动接收数据(并自动通过完成端口通知我们么??).

现在还剩下线程`ThreadPoolFunc`的逻辑, 分析其逻辑前, 先来看几个其中要用到的宏:  
1. `#define CONTAINING_RECORD(memberAddress, classType, memmer) ( (classType *) ( (PCHAR) (memberAddress) - (ULONG_PTR) (& (((classType *)0)->memmer) ) ) )`
用类成员变量的地址减去成员变量在类内的地址偏移, 得到类地址.  
2. 
```
BEGIN_IO_MSG_MAP()
    IO_MESSAGE_HANDLER(IORead, OnClientReading)
    IO_MESSAGE_HANDLER(IOWrite, OnClientWriting)
    IO_MESSAGE_HANDLER(IOInitialize, OnClientInitializing)
END_IO_MSG_MAP()
```
展开:
```
public:
    bool ProcessIOMessage(IOType clientIO, ClientContext* pContext, DWORD dwSize = 0)
    {
        bool bRet = false; 

        //////////////////////////////////////////
        if (IORead == clientIO)
            bRet = OnClientReading(pContext, dwSize); 

        //////////////////////////////////////////
        if (IOWrite == clientIO)
            bRet = OnClientWriting(pContext, dwSize); 

        //////////////////////////////////////////
        if (IOInitialize == clientIO)
            bRet = OnClientInitializing(pContext, dwSize); 

        //////////////////////////////////////////
        return bRet;
    }
```
这两个都是在`ThreadPoolFunc`需要用到的东西, 现在来看本体的逻辑:
```
CIOCPServer::ThreadPoolFunc(LPVOID thisContext)
{
    CIOCPServer* pThis = reinterpret_cast<CIOCPServer*>(thisContext);   //直接用this不是更好么...////线程函数是静态函数
    HANDLE hCompletionPort = pThis->m_hCompletionPort;
    LPOVERLAPPED lpOverlapped;
    ClientContext* lpClientContext;
    OVERLAPPEDPLUS*	pOverlapPlus;

    while(valid && not Shutdown)    
    {
        pOverlapPlus = NULL;
        lpClientContext = NULL;
        bError = false;
        bEnterRead = false;

        BOOL bIORet = GetQueuedCompletionStatus(hCompletionPort, &dwIoSize, (LPDWORD)&lpClientContext, &lpOverlapped, INFINITE);    //等待IO完成端口的 "事件?".

        DWORD dwIOError = GetLastError();

        pOverlapPlus = CONTAINING_RECORD(lpOverlapped, OVERLAPPEDPLUS, m_ol);   //由IO完成端口返回的LPOVERLAPPED地址来计算出存储该变量的OVERLAPPEDPLUS的地址, OVERLAPPEDPLUS存储着客户端操作类型.

        if (!bIORet && dwIOError != WAIT_TIMEOUT)   //错误处理, 关掉相应连接
        {
            if (lpClientContext && pThis->m_bTimeToKill == false)
            {
                pThis->RemoveStaleClient(lpClientContext, FALSE);
            }
            continue;
            bError = true;
        }
        if (!bError)    //没有错误的话, 控制一下资源使用??, 超限则标记错误.  
        {
            // Allocate another thread to the thread Pool?
            if (nBusyThreads == pThis->m_nCurrentThreads)
            {
                if (nBusyThreads < pThis->m_nThreadPoolMax)
                {
                    if (pThis->m_cpu.GetUsage() > pThis->m_nCPUHiThreshold)
                    {
                        UINT nThreadID = -1;
                    }
                }
            }
            // Thread timed out - IDLE?
            if (!bIORet && dwIOError == WAIT_TIMEOUT)
            {
                if (lpClientContext == NULL)
                {
                    if (pThis->m_cpu.GetUsage() < pThis->m_nCPULoThreshold)
                    {
                        // Thread has no outstanding IO - Server hasn't much to do so die
                        if (pThis->m_nCurrentThreads > pThis->m_nThreadPoolMin)
                            bStayInPool = FALSE;
                    }

                    bError = true;
                }
            }
        }
        if (!bError)
        {
            if (bIORet && NULL != pOverlapPlus && NULL != lpClientContext)
            {
                try
                {
                    pThis->ProcessIOMessage(pOverlapPlus->m_ioType, lpClientContext, dwIoSize); //执行OnClientReading(lpClientContext, dwIoSize)、
                    OnClientWriting(lpClientContext, dwIoSize)或者
                    OnClientInitializing(pContext, dwSize);
                }
                catch (...) {}
            }
        }
    }
}
```
总结一下, ThreadPoolFunc虽然看起来代码很多, 但核心代码很少, 主要逻辑就是:
1. 等待IO完成端口发来事件;
2. 取得事件类型, 并调用相应逻辑处理.
