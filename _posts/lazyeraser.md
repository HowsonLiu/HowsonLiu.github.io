---
title: lazyeraser
date: 2020-04-19 14:07:12
tags: 
    - chrome
    - duilib
---
谷歌`nbase`库提供了一种集合元素删除的方法，名为`LazyEraser`。
# 优点
C++容器类一般都是调用`erase`方法删除元素，但是`erase`会导致迭代器失效。`LazyEraser`延迟了删除元素的时机，一次删除，提高了效率，控制了删除的时机
# 抽象基类
```c++
class LazyEraser
{
public:
	LazyEraser() : lazy_erase_(0) {}
	/* NOTE: calling 'ApplyLasyErase' increases the reference count, calling 'ReleaseLasyErase' decreases the reference count */
	int AquireLazyErase() { return ++lazy_erase_; }
	int ReleaseLazyErase() { assert(lazy_erase_ > 0); return --lazy_erase_; }
	virtual void Compact() = 0;

protected:
	int lazy_erase_;
};
```
基类`LazyEraser`维护了一个引用计数。定义了抽象方法`Compact`，用于主动删除元素
# 例子
`nbase`库里记录观察者的列表类`ObserverList`就是使用`LazyEraser`的一个例子
```c++
template<typename ObserverType>
class ObserverList : public LazyEraser
{
public:
	void AddObserver(ObserverType *observer)
	{
		assert(observer);
		if (std::find(observers_.begin(), observers_.end(), observer) == observers_.end())
			observers_.push_back(observer);
	}

    // ...

	ObserverType* GetObserver(size_t index) const
	{
		if (index >= GetObserverCount())
			return NULL;
		return observers_[index];
	}

	size_t GetObserverCount() const { return observers_.size(); };

private:
	std::vector<ObserverType *> observers_;
};
```
本质上`ObserverList`属于`std::vector`，在插入、取值、计数等方面与`std::vector`表现一致，区别在于删除方面。
```c++
void RemoveObserver(ObserverType *observer)
{
	typename std::vector<ObserverType *>::iterator pos = std::find(observers_.begin(), observers_.end(), observer);
	if (pos == observers_.end())
		return;
	/* this method may be called within a traversal of the list */
	if (lazy_erase_ > 0)    // 关键
		*pos = NULL;
	else
		observers_.erase(pos);
}
```
调用`RemoveObserver`删除时如果此时引用计数不为0，那么容器不会调用`std::vector::erase`函数，只是仅仅将该地址置空。也就是说它并不会立即回收删除对象在容器里对应的内存（所以对应的迭代器还能正常使用），等到调用`Compact`函数，才会正常释放内存
```c++
void Compact()
{
	if (lazy_erase_ > 0 || observers_.empty())
		return;
	ObserverType **src = &observers_[0];
	ObserverType **target = src;
	ObserverType **end = src + observers_.size();
		/* fast compacting algorithm */
	while (src < end)
	{
		if (*src != NULL)
		{
			if (target != src)
				*target = *src;
			target++;
		}
		src++;
	}
		observers_.resize(target - &observers_[0]);
}
```
# 改进
如果当引用计数为0的时候忘了调用`Compact`，那么岂不是会造成内存泄漏？这部分内存得等到析构的时候才能正常释放？
谷歌考虑到这种情况，使用`RAII`机制新增了`AutoLazyEraser`类
```c++
class AutoLazyEraser
{
public:
	AutoLazyEraser(LazyEraser* lazy_eraser) : ptr_(lazy_eraser)
	{
		if (ptr_)
			ptr_->AquireLazyErase();
	}

	AutoLazyEraser(AutoLazyEraser &lazy_eraser)
	{
		if (lazy_eraser.ptr_)
			lazy_eraser.ptr_->AquireLazyErase();
		ptr_ = lazy_eraser.ptr_;
	}

	void operator=(AutoLazyEraser &lazy_eraser)
	{
		if (lazy_eraser.ptr_)
			lazy_eraser.ptr_->AquireLazyErase();
		ptr_ = lazy_eraser.ptr_;
	}

	~AutoLazyEraser()
	{
		if (ptr_)
		{
			if (ptr_->ReleaseLazyErase() < 1)
				ptr_->Compact();
		}
	}

private:
	LazyEraser *ptr_;
};
```
`AutoLazyEraser`类在析构时会自动调用一次`Compact`，这样就不怕泄露了