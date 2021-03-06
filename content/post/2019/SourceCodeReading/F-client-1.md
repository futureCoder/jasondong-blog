---
title: "F源码分析-1"
date: 2019-04-08T13:02:46+08:00
hierarchicalCategories: true
draft: false
categories: 
- 分类
- 技术文章
---
启动项选为Flamingo, F10单步调试启动, 程序从_tWinMain开始启动, 一步步跟下来, 发现程序在Startup.cpp::run()中的dlgMain.Create阻塞住了, 选择全部中断之后只筛出一个用户线程, 发现其实是阻塞在了CLoginDlg::DoModal()中, 并且堆栈信息显示确实是从Startup.cpp::run()调用过来的. 而DoModal再往下跟就是系统封好的东西了对于分析客户端流程也没啥意义. 那就从最后一个有意义的实例类CLoginDlg看起. 单单从表现上看, 这时应该客户端只做了一些微小的初始化工作, 然后显示出登录界面, 那么这时如果往下走的话唯一应该关心的东西应该就是点击登录按钮后执行的逻辑. 查看CLoginDlg定义, 很容易找到其中定义了一个消息处理回调```OnBtn_Login```, 这个函数逻辑是先在客户端做合法性检查. 把用户名密码保存到成员变量, 再开启一个线程```LoginThreadProc```并把自身实例作为参数传给线程, 


```
bool CIUSocket::Connect(int timeout /*= 3*/)
{
    Close();    //关闭现有连接
    m_hSocket = ::socket(AF_INET, SOCK_STREAM, 0);
    if (m_hSocket == INVALID_SOCKET)
        return false;
    long tmSend = 3 * 1000L;
    long tmRecv = 3 * 1000L;
    long noDelay = 1;
    setsockopt(m_hSocket, IPPROTO_TCP, TCP_NODELAY, (LPSTR)&noDelay, sizeof(long)); //禁用Nagle算法, 立即发送
    setsockopt(m_hSocket, SOL_SOCKET, SO_SNDTIMEO, (LPSTR)&tmSend, sizeof(long));   //设发送超时时间
    setsockopt(m_hSocket, SOL_SOCKET, SO_RCVTIMEO, (LPSTR)&tmRecv, sizeof(long));   //设接收超时时间
    unsigned long on = 1;
    if (::ioctlsocket(m_hSocket, FIONBIO, &on) == SOCKET_ERROR) //将socket设置成非阻塞的
        return false;
    struct sockaddr_in addrSrv = { 0 };
    struct hostent* pHostent = NULL;
    unsigned int addr = 0;
    if ((addrSrv.sin_addr.s_addr = inet_addr(m_strServer.c_str())) == INADDR_NONE)  //是否是点分十进制
    {
        pHostent = ::gethostbyname(m_strServer.c_str());    //不是点分十进制就按域名处理
        if (!pHostent)
        {
            LOG_ERROR("Could not connect server:%s, port:%d.", m_strServer.c_str(), m_nPort);
            return false;
        }
        else
            addrSrv.sin_addr.s_addr = *((unsigned long*)pHostent->h_addr);
    }

    addrSrv.sin_family = AF_INET;
    addrSrv.sin_port = htons((u_short)m_nPort);
    int ret = ::connect(m_hSocket, (struct sockaddr*)&addrSrv, sizeof(addrSrv));    //连接服务器, 非阻塞
    if (ret == 0)
    {
        LOG_INFO("Connect to server:%s, port:%d successfully.", m_strServer.c_str(), m_nPort);
        m_bConnected = true;
        return true;
    }
    else if (ret == SOCKET_ERROR && WSAGetLastError() != WSAEWOULDBLOCK)
    {
        LOG_ERROR("Could not connect to server:%s, port:%d.", m_strServer.c_str(), m_nPort);
        return false;
    }

    fd_set writeset;
    FD_ZERO(&writeset);
    FD_SET(m_hSocket, &writeset);
    struct timeval tv = { timeout, 0 };
    if (::select(m_hSocket + 1, NULL, &writeset, NULL, &tv) != 1)
    {   //超时和其他错误处理
        LOG_ERROR("Could not connect to server:%s, port:%d.", m_strServer.c_str(), m_nPort);
        return false;
    }
    m_bConnected = true;
    return true;
}
```
CIUSocket::Connect只做一件事就是连接chatserver, 上来先创建socket并将其设置为非阻塞模式, 禁用Nagle算法并设置收/发超时时间为3s, 

主线程中 CMainDlg::OnLoginResult() 初始化CFlamingoClient中所有的网络线程InitNetThreads(), 之后调用CFlamingoClient::GetFriendList()向消息发送线程m_SendMsgThread加入一条待发送数据. 

看一下线程初始化部分, 即CFlamingoClient::InitNetThreads:
```
bool CFlamingoClient::InitNetThreads()
{
    CIUSocket::GetInstance().SetRecvMsgThread(&m_RecvMsgThread);    //有个疑问, 在Init之前先设置了RecvMsgThread, 为什么呢? 暂且先不管, 一会儿再看.
    CIUSocket::GetInstance().Init();

    m_SendMsgThread.Start();
    m_RecvMsgThread.Start();

    m_SendMsgThread.m_lpFMGClient = this;

    m_FileTask.Start();
    m_ImageTask.Start();

    //CSendFileThread::GetInstance().AttachSocketClient(&m_SocketClient);
    //CSendFileThread::GetInstance().Start();
    return true;
}
```

大概过一下CFlamingoClient的定义, 可以看到这是一个单例类, 并且从名字可以推测这是封装客户端逻辑操作的(因为UI逻辑封装在了CMainDlg中), 那么CFlamingoClient暂且算作业务逻辑层, 他里面有四个线程对象:
1. CSendMsgThread m_SendMsgThread
2. CRecvMsgThread m_RecvMsgThread
3. CFileTaskThread m_FileTask
4. CImageTaskThread m_ImageTask
这四个线程对象继承自同一个父类CThread, CThread的声明定义很短, 直接展开来看一下:
```
class CThread
{
public:
    CThread(){};
    virtual ~CThread(){};
    CThread(const CThread& rhs) = delete;
    CThread& operator=(const CThread& rhs) = delete;
    void Start()
    {
        if (!m_spThread)
            m_spThread.reset(new std::thread(std::bind(&CThread::ThreadProc, this)));
    }
    virtual void Stop() = 0;
    void Join()
    {
        if (m_spThread && m_spThread->joinable())
            m_spThread->join();
    }
protected:
    virtual void Run() = 0;
private:
    void ThreadProc()
    {
        Run();
    }
protected:
    bool                            m_bStop{ false };
private:
    std::shared_ptr<std::thread>    m_spThread;
};
```
CThread使用很简单: 子类实现Run()和Stop(), 其中Run()是线程函数, 线程退出时需要做的一些清理操作放在Stop()中. 通过实例对象的Start()方法启动线程, std::bind将CThread的成员函数ThreadProc()和隐含参数this绑定并将绑定后的函数作为线程函数作为std::thread构造函数的参数并实例化(new)一个std::thread对象, 这个对象保存在只能指针m_spThread中. 线程启停的大致逻辑清楚了, 接来下具体看下这四个线程函数逻辑. 

```
void CSendMsgThread::Run()
{
    while (!m_bStop)
    {
        CNetData* lpMsg;        //指向CNetData的指针变量, CNetData是客户端所有协议包的父类.
        {
            std::unique_lock<std::mutex> guard(m_mtItems);
            while (m_listItems.empty())
            {
                if (m_bStop)
                    return;
                m_cvItems.wait(guard);      //阻塞在条件变量上
            }
            lpMsg = m_listItems.front();    //拿到协议包
            m_listItems.pop_front();        //消费
        }   //guard出作用域, 析构解锁.
        HandleItem(lpMsg);                  //根据协议包的类型(m_uType)进行处理, 注意这里已经不是互斥区了.
    }
}
```
HandleItem在函数内外都没有加锁保护, 所以是非线程安全的. 不过实际上只有线程函数中调用了他. 这个函数逻辑如下:
1. 根据协议包的类型执行具体的协议包处理逻辑;
2. 在具体的协议包处理逻辑中, 将协议包用BinaryWriteStream序列化到一个string中;
3. 由CIUSocket::Send(const std::string& strBuffer)负责将序列化后的string发送出去;
4. 包序列号自增;
5. 析构协议包.




CIUSocket 作为网络通信层, 其中有两个线程:
1. m_spSendThread 数据发送线程, 线程函数 CIUSocket::SendThreadProc
2. m_spRecvThread 数据接收线程, 线程函数 CIUSocket::RecvThreadProc
首先看数据发送线程的逻辑:
```
void CIUSocket::SendThreadProc()
{
    while (!m_bStop)    //m_bStop在CIUSocket::Uninit()置为true, Uninit调用时机为: 1. 网络连接断开; 2. 直界面执行OnDestroy, FMGClient进行清理时调用; 3. 
    {
        std::unique_lock<std::mutex> guard(m_mtSendBuf);    //对sendBuffer上锁
        while (m_strSendBuf.empty())    //如果sendBuffer中没有数据
        {
            if (m_bStop)
                return;
            m_cvSendBuf.wait(guard);    //阻塞自身, 等待sendBuffer的条件变量
        }

        if (!Send())    //到这里说明m_strSendBuf中有数据可以发送了;
        {
            // -- 进行重连，如果连接不上，则向客户报告错误
        }
    }
}

bool CIUSocket::Send()
{
    //-- 如果未连接则重连，重连也失败则返回FALSE
    //-- TODO: 在发送数据的过程中重连没什么意义，因为与服务的Session已经无效了，换个地方重连
    if (IsClosed() && !Connect())   //判断连接是否断开, 断开则进行重连
    {   //重连失败
        LOG_ERROR("connect server:%s:%d error.", m_strServer.c_str(), m_nPort);
        return false;
    }

    int nSentBytes = 0;
    int nRet = 0;
    while (true)
    {
        nRet = ::send(m_hSocket, m_strSendBuf.c_str(), m_strSendBuf.length(), 0);
        if (nRet == SOCKET_ERROR)
        {
            if (::WSAGetLastError() == WSAEWOULDBLOCK)
                break;
            else
            {
                LOG_ERROR("Send data error, disconnect server:%s, port:%d.", m_strServer.c_str(), m_nPort);
                Close();
                return false;
            }
        }
        else if (nRet < 1)
        {
            //-- 一旦出现错误就立刻关闭Socket
            LOG_ERROR("Send data error, disconnect server:%s, port:%d.", m_strServer.c_str(), m_nPort);
            Close();
            return false;
        }

        m_strSendBuf.erase(0, nRet);    //把已经发出去的数据清掉
        if (m_strSendBuf.empty())       //如果数据发送完毕
            break;

        ::Sleep(1);
    }

    {
        std::lock_guard<std::mutex> guard(m_mutexLastDataTime);
        m_nLastDataTime = (long)time(NULL);     //互斥修改最后一次进行通信的时间戳
    }
    return true;
}
```
注意, CIUSocket::Send()本身是非线程安全的, 对m_strSendBuf的访问并没有做互斥, 如果在多线程中调用该函数, 线程安全由调用者来保证. 查看一下Send()的引用发现也确实是这么做的, 在工程中只有CIUSocket::SendThreadProc()内会调用Send(), 而SendThreadProc()在线程函数内对m_strSendBuf的访问加了锁, 锁的作用域是整个线程函数循环体. 这里推测一下, 因为Send()本身是非线程安全, 若是有意设计, 那么这个函数的设计目的就是为了某个特定线程封装操作逻辑接口, 这个特定线程就是CIUSocket::SendThreadProc(), 该线程是m_cvSendBuf唯一的消费者. CIUSocket::SendThreadProc()函数体在上面做了简单分析, 逻辑很简单: 查看是否可消费, 如果可以的话就调用Send()进行消费并一次消费干净(过程中消费对象会上锁, 生产者将无法将产品(待消费对象)放入); 如果不可消费则阻塞在条件变量上, 等待生产者唤醒. 这里消费对象就是m_cvSendBuf中的数据. 

 那么生产者是谁呢, 搜索发现只有CIUSocket::Send(const std::string& strBuffer)向m_cvSendBuf中添加数据, 那么该函数是唯一的生产接口, 该接口的调用者就是生产者. 查找发现有两个生产者: 
 1. 线程函数CIUSocket::RecvThreadProc(); 
 2. 线程函数CSendMsgThread::Run(). 接下来看一下这两个线程的逻辑.
 ```
 void CIUSocket::RecvThreadProc()   //接收数据包线程函数
 {
    int nRet;
    //上网方式 
    DWORD   dwFlags;
    BOOL    bAlive;
    while (!m_bStop)
    {
        //检测到数据则收数据
        nRet = CheckReceivedData();     //该函数内通过select检测m_hSocket是否可读或异常, 超时时间为500ms
        //出错
        if (nRet == -1)
        {
            m_pRecvMsgThread->NotifyNetError(); //通知主窗口处理网络连接错误
        }
        //无数据
        else if (nRet == 0)     //这里的0不是recv的返回值而是CheckReceivedData的, 表明500ms没有检测到m_hSocket可读. 
        {
            bAlive = ::IsNetworkAlive(&dwFlags);		//是否在线
            if (!bAlive && ::GetLastError() == 0)
            {
                //网络已经断开
                m_pRecvMsgThread->NotifyNetError();
                LOG_ERROR("net error, exit recv and send thread...");
                Uninit();       //断开所有连接
                break;
            }

            long nLastDataTime = 0;
            {
                std::lock_guard<std::mutex> guard(m_mutexLastDataTime);
                nLastDataTime = m_nLastDataTime;        //用临时变量缩小锁粒度
            }

            if (m_nHeartbeatInterval > 0)
            {
                if (time(NULL) - nLastDataTime >= m_nHeartbeatInterval)     //一段时间内都没收到数据就发个心跳包
                    SendHeartbeatPackage();     //发送心跳包, 正是这个函数中调用了CIUSocket::Send(const std::string& strBuffer)向m_cvSendBuf中生产数据
            }
        }
        //-- 有数据
        else if (nRet == 1)
        {
            if (!Recv())    //接收数据, 下面分析
            {
                m_pRecvMsgThread->NotifyNetError();
                continue;
            }

            DecodePackages();   //解包
        }// end if
    }// end while-loop
 }

bool CIUSocket::Recv()  
{
    int nRet = 0;
    char buff[10 * 1024];
    while (true)
    {
        nRet = ::recv(m_hSocket, buff, 10 * 1024, 0);
        if (nRet == SOCKET_ERROR)				//-- 一旦出现错误就立刻关闭Socket
        {
            if (::WSAGetLastError() == WSAEWOULDBLOCK)
                break;
            else
            {
                LOG_ERROR("Recv data error, errorNO=%d.", ::WSAGetLastError());
                //Close();
                return false;
            }
        }
        else if (nRet < 1)
        {
            LOG_ERROR("Recv data error, errorNO=%d.", ::WSAGetLastError());
            //Close();
            return false;
        }
        m_strRecvBuf.append(buff, nRet);        //注意: m_strRecvBuf 的访问没有加任何保护, 说明Recv只能被单个线程调用. 而它的调用者RecvThreadProc也没有任何加保护的地方, 加之这里只生产数据而并不消费, 可以断定消费m_strRecvBuf的也是RecvThreadProc线程并且生产和消费一定是有关联关系的两个操作节点.
        ::Sleep(1);
    }
    {
        std::lock_guard<std::mutex> guard(m_mutexLastDataTime);
        m_nLastDataTime = (long)time(NULL);
    }
    return true;
}
bool CIUSocket::DecodePackages()    //解包, 
{
    //-- 一定要放在一个循环里面解包，因为可能一片数据中有多个包，
    //-- 对于数据收不全，这个地方我纠结了好久T_T
    while (true)
    {
        //-- 接收缓冲区不够一个包头大小
        if (m_strRecvBuf.length() <= sizeof(msg))
            break;

        msg header;
        memcpy_s(&header, sizeof(msg), m_strRecvBuf.data(), sizeof(msg));
        //数据压缩过
        if (header.compressflag == PACKAGE_COMPRESSED)
        {
            //防止包头定义的数据是一些错乱的数据，这里最大限制每个包大小为10M
            if (header.compresssize >= MAX_PACKAGE_SIZE || header.compresssize <= 0 ||
                header.originsize >= MAX_PACKAGE_SIZE || header.originsize <= 0)
            {
                LOG_ERROR("Recv a illegal package, compresssize: %d, originsize=%d.", header.compresssize, header.originsize);
                m_strRecvBuf.clear();   //非法数据包直接清掉, 
                return false;
            }

            //-- 接收缓冲区不够一个整包大小（包头+包体）
            if (m_strRecvBuf.length() < sizeof(msg) + header.compresssize)  //还没收全, 继续收.
                break;

            //-- 去除包头信息
            m_strRecvBuf.erase(0, sizeof(msg));             //清掉包头
            //-- 拿到包体
            std::string strBody;
            strBody.append(m_strRecvBuf.c_str(), header.compresssize);      //拿出整个包体
            //-- 去除包体信息
            m_strRecvBuf.erase(0, header.compresssize);     //清掉包体

            //-- 解压
            std::string strUncompressBody;
            if (!ZlibUtil::UncompressBuf(strBody, strUncompressBody, header.originsize))
            {
                LOG_ERROR("uncompress buf error, compresssize: %d, originsize: %d", header.compresssize, header.originsize);
                m_strRecvBuf.clear();       //解压发生错误, 清掉所有已收数据.  这里有疑问. 难道这不会破坏正常包么??
                return false;
            }

            m_pRecvMsgThread->AddMsgData(strUncompressBody);        //调用另一个线程对象的接口, 向该线程放入数据
        }
        //-- 数据未压缩过
        else
        {
            //-- 防止包头定义的数据是一些错乱的数据，这里最大限制每个包大小为10M
            if (header.originsize >= MAX_PACKAGE_SIZE || header.originsize <= 0)
            {
                LOG_ERROR("Recv a illegal package, originsize=%d.", header.originsize);
                m_strRecvBuf.clear();       //非法数据包直接清掉, 
                return false;
            }

            //-- 接收缓冲区不够一个整包大小（包头+包体）
            if (m_strRecvBuf.length() < sizeof(msg) + header.originsize)        //还没收全, 继续收.
                break;

            //-- 去除包头信息
            m_strRecvBuf.erase(0, sizeof(msg));
            //-- 拿到包体
            std::string strBody;
            strBody.append(m_strRecvBuf.c_str(), header.originsize);
            //-- 去除包体信息
            m_strRecvBuf.erase(0, header.originsize);
            m_pRecvMsgThread->AddMsgData(strBody);                  //调用另一个线程对象的接口, 向该线程放入数据
        }
    }// end while
    return true;
}
```

 总结一下这两个线程:
 1. 网络数据发送线程: m_spSendThread, 线程函数是CIUSocket::SendThreadProc, 职责是将其他模块或线程投递过来的数据正确进行发送. m_spSendThread是m_cvSendBuf唯一的消费者, m_spRecvThread 和 m_SendMsgThread 是m_cvSendBuf唯二的生产者, 二者调用同一个投送数据的接口, 数据投送完毕后通过条件变量通知m_spSendThread. m_spRecvThread 投送的是心跳包, m_SendMsgThread投送的是经由HandleItem(CNetData* pNetData)工厂生产包装后的协议数据.
 2. 网络数据接收线程: m_spRecvThread, 线程函数是CIUSocket::RecvThreadProc, 职责是按包(协议)确定数据边界, 读取socket接收的数据并进行解包, 最后将包体数据经由CRecvMsgThread::AddMsgData(const std::string& pMsgData)投送到 m_pRecvMsgThread 线程并通过条件变量通知m_pRecvMsgThread进行消费. 
再谈一下这块儿的设计, CUISocket作为一个单例类可以在全局使用, 数据发送线程m_spSendThread 和 数据接收线程m_spRecvThread 同属CIUSocket类, 在这一层保证网络数据的正确发送与接收: 将其他模块投送过来的协议包进行压缩并发送, 将socket收到的网络数据进行解压并转成协议包.

