---
title: "F源码分析-1"
date: 2019-04-08T13:02:46+08:00
hierarchicalCategories: true
draft: false
categories: 
- 分类
- 技术文章
---


CFlamingoClient持有


CIUSocket
CIUSocket::Init()

CIUSocket::Connect()

CSendMsgThread::Run()

CRecvMsgThread::Run()

CFileTaskThread::Run()

CImageTaskThread::Run()

CIUSocket::CheckReceivedData()

_tWinMain
atlapp.h::1271

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

主线程中 CMainDlg::OnLoginResult() 初始化CFlamingoClient中所有的网络线程InitNetThreads(), 之后调用CFlamingoClient::GetFriendList()向消息发送线程m_SendMsgThread加入一条待发送数据, ~~并开启CCheckNetworkStatusTask线程, CCheckNetworkStatusTask在线程函数中每3秒检测一次网络状态并将状态由FMG_MSG_NETWORK_STATUS_CHANGE事件抛出, ~~

看一下网络线程的初始化部分
```
bool CFlamingoClient::InitNetThreads()
{
    CIUSocket::GetInstance().SetRecvMsgThread(&m_RecvMsgThread);    //
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
在Init之前先设置了RecvMsgThread, 为什么呢? 

CFlamingoClient 算作业务逻辑层, 里面有四个线程:
1. 


CIUSocket 作为网络通信层, 其中有两个线程:
1. m_spSendThread 数据发送线程, 线程函数 CIUSocket::SendThreadProc
2. m_spRecvThread 数据接收线程, 线程函数 CIUSocket::RecvThreadProc
首先看数据发送线程的逻辑:
```
CIUSocket::SendThreadProc()
{
    while (!m_bStop)    //m_bStop在CIUSocket::Uninit()置为true, Uninit调用时机为: 1. 网络连接断开; 2. 直界面执行OnDestroy, FMGClient进行清理时调用; 3. 
    {
        std::unique_lock<std::mutex> guard(m_mtSendBuf);
        while (m_strSendBuf.empty())
        {
            if (m_bStop)
                return;

            m_cvSendBuf.wait(guard);
        }

        if (!Send())
        {
            //进行重连，如果连接不上，则向客户报告错误
        }
    }
}
```