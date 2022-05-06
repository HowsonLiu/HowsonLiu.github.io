---
title: thread_local
date: 2020-04-16 22:43:30
tags:
---
这个是谷歌nbase库里面的线程局部存储模板类
# 跨平台
```c++
struct ThreadLocalPlatform
{
#if defined(OS_WIN)
	typedef unsigned long SlotType;
#elif defined(OS_POSIX)
	typedef pthread_key_t SlotType;
#endif

	static void AllocateSlot(SlotType &slot);
	static void FreeSlot(SlotType &slot);
	static void* GetValueFromSlot(SlotType &slot);
	static void SetValueInSlot(SlotType &slot, void *value);
};

// unix
void ThreadLocalPlatform::AllocateSlot(SlotType &slot)
{
	int error = pthread_key_create(&slot, NULL);
	assert(error == 0);
}

// win
void ThreadLocalPlatform::AllocateSlot(SlotType &slot)
{
	slot = ::TlsAlloc();
	assert(slot != TLS_OUT_OF_INDEXES);
}
```
这个类提供跨平台API选择，Windows主要通过`::TlsAlloc`实现，unix主要通过`pthread_key_create`实现

# 模板
```c++
template<typename Type>
class ThreadLocalPointer
{
public:

	ThreadLocalPointer() : slot_()
	{
		internal::ThreadLocalPlatform::AllocateSlot(slot_);
	}

	~ThreadLocalPointer()
	{
		internal::ThreadLocalPlatform::FreeSlot(slot_);
	}

	Type* Get()
	{
		return static_cast<Type*>(internal::ThreadLocalPlatform::GetValueFromSlot(slot_));
	}

	void Set(Type *ptr)
	{
		internal::ThreadLocalPlatform::SetValueInSlot(slot_, ptr);
	}

private:
	typedef internal::ThreadLocalPlatform::SlotType SlotType;
	SlotType slot_;

	DISALLOW_COPY_AND_ASSIGN(ThreadLocalPointer);
};
```
- 很简单的贫血模式
- 对目标类并不直接存储，而是记录其地址
- 禁止拷贝，保证安全