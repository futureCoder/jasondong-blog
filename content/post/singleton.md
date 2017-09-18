---
title: "设计模式-单例模式"
date: 2017-09-11T09:47:29+08:00
draft: true
---

####单例模式
单例模式是一种常用的软件设计模式。在应用这个模式时，单例对象的类必须保证只有一个实例存在。许多时候整个系统只需要拥有一个的全局对象，这样有利于我们协调系统整体的行为。比如在某个服务器程序中，该服务器的配置信息存放在一个文件中，这些配置数据由一个单例对象统一读取，然后服务进程中的其他对象再通过这个单例对象获取这些配置信息。这种方式简化了在复杂环境下的配置管理。
实现单例模式的思路是：一个类能返回对象一个指针/引用(永远是同一个)和一个获得该实例的方法（必须是静态方法，通常使用getInstance这个名称）；当我们调用这个方法时，如果类持有的指针/引用不为空就返回这个指针/引用，如果类保持的指针/引用为空就创建该类的实例并将实例的指针/引用赋予该类保持的指针/引用；同时我们还将该类的构造函数定义为私有方法，这样其他处的代码就无法通过调用该类的构造函数来实例化该类的对象，只有通过该类提供的静态方法来得到该类的唯一实例。
单例模式在多线程的应用场合下必须小心使用。如果当唯一实例尚未创建时，有两个线程同时调用创建方法，那么它们同时没有检测到唯一实例的存在，从而同时各自创建了一个实例，这样就有两个实例被构造出来，从而违反了单例模式中实例唯一的原则。 解决这个问题的办法是为指示类是否已经实例化的变量提供一个互斥锁(虽然这样会降低效率)。

单例模式算是设计模式中最容易理解，也是最容易手写代码的模式。但其中的坑却不少，所以常作为面试题来考。本文主要对几种单例的写法进行整理，并分析其优缺点。
####构造方法
#####懒汉方式 指全局的单例实例在第一次被使用时构建，线程不安全
{{< codeblock "懒汉方式" >}}
template<class T>
public class Singleton
{
private:
	static T* m_ptr;
	Singleton(){}
public:
	static inline T& Instance(){return *InstancePtr();}
	static inline T* InstancePtr()
	{
		if (nullptr == m_ptr)
		{
			m_ptr = new T();
		}
		return m_ptr;
	}
}
{{< /codeblock >}}
一目了然，懒汉方式是用到时候才创建类，代码简单但却存在严重问题。当多个线程并行执行```InstancePtr```时，若线程1和线程2同时执行到了```if (nullptr == m_ptr)```,那么便会创建多个该类的实例，违反了该类为单例类的原则。

#####饿汉方式 指全局的单例实例在类装载时构建，线程安全
{{< codeblock "饿汉方式" >}}
template<class T>
public class Singleton
{
private:
	static T* m_ptr;
	Singleton(){}
public:
	static inline T& Instance(){return *InstancePtr();}
	static inline T* InstancePtr()
	{
		return m_ptr;
	}
}
template<class T>
T* Singleton<T>::m_ptr = new T();
{{< /codeblock >}}

{{< codeblock "Singleton.h" >}}
#include <stdlib.h>
#include "DLMalloc.h"

#if defined(__LINUX__) || defined(__linux__) || defined (LINUX)
	#define nullptr 0
#endif

namespace Galaxy
{
	enum MemType
	{
		MT_UNKNOWN = 0,
		MT_HEAP_NEW,
		MT_HEAP_MALLOC,
		MT_STACK,
	};

	template<class T>
	class Singleton
	{
	public:
		Singleton(){}
		virtual ~Singleton(){}

		static inline T& Instance(){return *InstancePtr();}
		static inline T* InstancePtr()
		{
			if (nullptr == m_ptr)
			{
				m_ptr = (T*)
#ifdef _Client
				::operator new
#else
				dlmalloc
#endif
				(sizeof(T));

				if (m_ptr)
				{
					m_type = MT_HEAP_MALLOC;
					new(m_ptr)T();
				}
			}
			return m_ptr;
		}
		// An interface to release the singleton object.
		// We must call this to prevent memory leak!
		static inline void ReleaseInstance()
		{
			if (nullptr == m_ptr)
				return;

			switch (m_type)
			{
				case MT_HEAP_NEW:
					delete m_ptr;
					break;
				case MT_HEAP_MALLOC:
					m_ptr->~T();
#ifdef _Client
					::operator delete
#else
					dlfree
#endif
					(m_ptr);
					break;
				default:
					break;
			}
			m_ptr = nullptr;
			m_type = MT_UNKNOWN;
		}
		static inline void SetInstance(T* pObj, MemType mt = MT_HEAP_NEW)
		{
			ReleaseInstance();
			if (nullptr != pObj)
			{
				m_ptr = pObj;
				m_type = mt;
			}
		}

	private:
		static T* m_ptr;
		static MemType m_type;
	};

	template<class T>
	T* Singleton<T>::m_ptr = nullptr;
	template<class T>
	MemType Singleton<T>::m_type = MT_UNKNOWN;
}
{{< /codeblock >}}