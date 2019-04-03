---
title: "gh0st源码分析-2"
date: 2019-03-30T21:04:41+08:00
hierarchicalCategories: true
draft: false
categories: 
- 分类
- 技术文章
---

Server端收发包逻辑:
在上文可知, 客户端数据全部封装在一个结构体ClientContext中, 这个结构大概长这个样子:
```
struct ClientContext
{
    SOCKET				m_Socket;               // 客户端socket句柄
	CBuffer				m_WriteBuffer;          // 要发送的数据缓冲, 在OnClientWriting中取出到m_wsaOutBuffer, 调用WSASend时候被使用
	CBuffer				m_CompressionBuffer;	// 接收到的压缩的数据
	CBuffer				m_DeCompressionBuffer;	// 解压后的数据
	CBuffer				m_ResendWriteBuffer;	// 上次发送的数据包，接收失败时重发时用
	int					m_Dialog[2];            // 放对话框列表用，第一个int是类型，第二个是CDialog的地址
	int					m_nTransferProgress;	//

	WSABUF				m_wsaInBuffer;
	BYTE				m_byInBuffer[8192];

	WSABUF				m_wsaOutBuffer;
	HANDLE				m_hWriteComplete;

	LONG				m_nMsgIn;
	LONG				m_nMsgOut;	

	BOOL				m_bIsMainSocket; // 是不是主socket

	ClientContext*		m_pWriteContext;
	ClientContext*		m_pReadContext;
};
```

简单看一下CBuffer封装的操作:
```
class CBuffer  
{
public:
	void ClearBuffer();		//清空数据, 但不会缩小Buffer所占内存(可能是引用的函数有bug).

	UINT Delete(UINT nSize);	//删除数据, nSize大于内存则返回, 大于数据区长度则取数据区长度,
	UINT Read(PBYTE pData, UINT nSize);		//从Buffer头部开始读nSize个字节至pData中, 然后在Buffer中删除读出的数据(通过MemMove把basePtr + nSize后的所有数据拷到basePtr实现...)
	BOOL Write(PBYTE pData, UINT nSize);	//在Buffer尾部追加pData
	BOOL Write(CString& strData);
	UINT GetBufferLen() { return null == m_pBase ? 0 :  m_pPtr - m_pBase; }     //实际数据字节数, 总内存的大小是GetMemSize()
	int Scan(PBYTE pScan,UINT nPos);		//从nPos位置起查找pScan字符串,找到则返回Buffer中匹配的字符串尾部偏移.
	BOOL Insert(PBYTE pData, UINT nSize);	//在Buffer头部插入pData
	BOOL Insert(CString& strData);

	void Copy(CBuffer& buffer);		//对另一个buffer进行深拷贝

	PBYTE GetBuffer(UINT nPos=0);	//

	void FileWrite(const CString& strFileName);		//写入文件
protected:
	UINT ReAllocateBuffer(UINT nRequestedSize);	//对nRequestedSize按1024对齐, VirtualAlloc新内存, VirtualFree原内存(nRequestedSize小于原Buffer内存大小则不执行)	--- 仅用于对Buffer扩容
	UINT DeAllocateBuffer(UINT nRequestedSize);	//对Buffer扩容, 但申请空间不足以容纳原Buffer数据或按1024对齐后比原Buffer占用内存小则不执行 (这个看起来是有bug的, 从被引用的功能上来看, 该函数应该具备紧缩Buffer的功能, 而目前的处理是只要传入的大小比Buffer本身的内存小就返回不执行了, 感觉可能判断大小的符号写反了...) 
	UINT GetMemSize() { return m_nSize; }		//分配内存的大小, 数据实际占用的大小是GetBufferLen()

protected:
	PBYTE	m_pBase;	//Buffer基址
	PBYTE	m_pPtr;		//Buffer数据尾地址
	UINT	m_nSize;	//Buffer总内存
}
```
上面介绍了CBuffer的所有操作, 可以看到其就是封装了对PBYTE(字节流)数据存取和内存分配等的相关操作, 除去看起来存在bug的DeAllocateBuffer不谈, 这个CBuffer本身的效率看起来并不高, 由于基地址不变, 导致经常会出现内存移动操作. 如果再新开一个变量记录数据头部和尾部, 形成**环形缓冲区**, 可以避免频繁的内存移动操作, 只有在缓冲区不足需要重分配时候才需要进行内存移动. 

```
bool CIOCPServer::OnClientReading(ClientContext* pContext, DWORD dwIoSize)
{
	CLock cs(CIOCPServer::m_cs, "OnClientReading");		//CLock构造函数加锁, 析构解锁
	try
	{
		static DWORD nLastTick = GetTickCount();	//注意:
        static DWORD nBytes = 0;					//这两个是static只会初始化一次
        nBytes += dwIoSize;		//接收了多少字节数据
		if (GetTickCount() - nLastTick >= 1000)		//
        {
            nLastTick = GetTickCount();
            InterlockedExchange((LPLONG)&(m_nRecvKbps), nBytes);	//计算实时速度
            nBytes = 0;
        }
		if (dwIoSize == 0)		//注意!!!查一下这个0代表什么, 底层是send或recv一类的么.. 0表示连接断开??
        {
            RemoveStaleClient(pContext, FALSE);
            return false;
        }
		if (dwIoSize == FLAG_SIZE && memcmp(pContext->m_byInBuffer, m_bPacketFlag, FLAG_SIZE) == 0)
        {
            // 重新发送
            Send(pContext, pContext->m_ResendWriteBuffer.GetBuffer(), pContext->m_ResendWriteBuffer.GetBufferLen());
            // 必须再投递一个接收请求
            PostRecv(pContext);
            return true;
        }
	}
}
```



```
void CIOCPServer::Send(ClientContext* pContext, LPBYTE lpData, UINT nSize)
{
	if (nSize > 0)
	{
		unsigned long	destLen = (double)nSize * 1.001 + 12;	//新版本的zlib应该是用compressBound计算出
		LPBYTE			pDest = new BYTE[destLen];				
		int	nRet = compress(pDest, &destLen, lpData, nSize);	//压缩待发送数据
		if (nRet != Z_OK)
		{
			delete[] pDest;
			return;
		}
		LONG nBufLen = destLen + HDR_SIZE;
		// 5 bytes packet flag
		pContext->m_WriteBuffer.Write(m_bPacketFlag, sizeof(m_bPacketFlag));
		// 4 byte header [Size of Entire Packet]
		pContext->m_WriteBuffer.Write((PBYTE)&nBufLen, sizeof(nBufLen));
		// 4 byte header [Size of UnCompress Entire Packet]
		pContext->m_WriteBuffer.Write((PBYTE)&nSize, sizeof(nSize));
		// Write Data
		pContext->m_WriteBuffer.Write(pDest, destLen);
		delete[] pDest;		//归还用来存储压缩数据的临时堆空间

		// 发送完后，再备份数据, 因为有可能是m_ResendWriteBuffer本身在发送,所以不直接写入
		// 否则在m_ResendWriteBuffer.ClearBuffer()之后, lpData将变为悬空指针
		LPBYTE lpResendWriteBuffer = new BYTE[nSize];
		CopyMemory(lpResendWriteBuffer, lpData, nSize);
		pContext->m_ResendWriteBuffer.ClearBuffer();
		pContext->m_ResendWriteBuffer.Write(lpResendWriteBuffer, nSize);	// 备份发送的数据
		delete[] lpResendWriteBuffer;
	}
	else
	{
		pContext->m_WriteBuffer.Write(m_bPacketFlag, sizeof(m_bPacketFlag));
		pContext->m_ResendWriteBuffer.ClearBuffer();
		pContext->m_ResendWriteBuffer.Write(m_bPacketFlag, sizeof(m_bPacketFlag));
	}

	WaitForSingleObject(pContext->m_hWriteComplete, INFINITE);	//无限等待m_hWriteComplete事件发生

}
```
