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


再看一下另外一个线程ThreadPoolFunc的逻辑.
