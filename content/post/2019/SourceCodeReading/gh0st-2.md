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
	int					m_nTransferProgress;

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
	void ClearBuffer();		//

	UINT Delete(UINT nSize);
	UINT Read(PBYTE pData, UINT nSize);		//从Buffer头部开始读nSize个字节至pData中, 然后在Buffer中删除读出的数据(通过MemMove把basePtr + nSize后的所有数据拷到basePtr实现...)
	BOOL Write(PBYTE pData, UINT nSize);	//在Buffer尾部追加pData
	BOOL Write(CString& strData);
	UINT GetBufferLen() { return null == m_pBase ? 0 :  m_pPtr - m_pBase; }     //实际数据字节数, 总内存的大小是GetMemSize()
	int Scan(PBYTE pScan,UINT nPos);
	BOOL Insert(PBYTE pData, UINT nSize);	//在Buffer头部插入pData
	BOOL Insert(CString& strData);

	void Copy(CBuffer& buffer);	

	PBYTE GetBuffer(UINT nPos=0);

	CBuffer();
	virtual ~CBuffer();

	void FileWrite(const CString& strFileName);
protected:
	UINT ReAllocateBuffer(UINT nRequestedSize);	//对nRequestedSize按1024对齐, VirtualAlloc新内存, VirtualFree原内存(nRequestedSize小于原Buffer内存大小则不执行)	--- 仅用于对Buffer扩容
	UINT DeAllocateBuffer(UINT nRequestedSize);	//对Buffer扩容, 但申请空间不足以容纳原Buffer数据或按1024对齐后比原Buffer占用内存小则不执行
	UINT GetMemSize() { return m_nSize; }		//分配内存的大小, 数据实际占用的大小是GetBufferLen()

protected:
	PBYTE	m_pBase;	//Buffer基址
	PBYTE	m_pPtr;		//Buffer数据尾地址
	UINT	m_nSize;	//Buffer总内存
}
```