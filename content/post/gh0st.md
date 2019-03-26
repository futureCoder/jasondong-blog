`CIniFile`读取配置文件, `CGh0stApp`在其重载的`InitInstance()`中, 调用主窗口`CMainFrame`的`Activate(UINT nPort, UINT nMaxConnections)`函数并将`CIniFile`中的配置传入.  
`CMainFrame::Activate`主要只做了一件事, 就是创建一个`CIOCPServer`的实例并且调用`CIOCPServer::Initialize(NOTIFYPROC pNotifyProc, CMainFrame* pFrame,  int nMaxConnections, int nPort)`进行初始化, 其中, **参数1**`pNotifyProc`是一个回调函数, 传入值是`CMainFrame::NotifyProc(LPVOID lpParam, ClientContext* pContext, UINT nCode)`函数,其作用是将一条`WM_UPDATE_MAINFRAME`消息放入消息队列并附加两个消息参数nCode和pContent.**参数2**是主窗口指针,**参数3**是最大连接数,**参数4**是监听端口.`Initialize`的前三个参数全被`CIOCPServer`存至成员变量,下面来介绍`CIOCPServer::Initialize`的逻辑.

1. 创建异步I/O套接字m_socListen
2. WSACreateEvent()创建事件对象,并使用WSAEventSelect事件选择模型,只关心FD_ACCEPT事件.
3. bind, listen没什么好说的, 之后创建并启动一个监听线程`ListenThreadProc`
4. 监听线程创建成功后, 