---
title: nbase 消息循环（一）MessagePump
date: 2020-04-20 22:00:01
categories: Duilib
tags:
	- chrome
	- duilib
description: chrome源码中的公共库nbase中的消息循环代码解析
---
`MessagePump`控制消息循环的开始与停止
```c++
class BASE_EXPORT MessagePump
{
public:
	class BASE_EXPORT Delegate
	{
	public:
		virtual ~Delegate() {}
		virtual bool DoWork() = 0;
		virtual bool DoDelayedWork(TimeTicks *next_delayed_message_time) = 0;
		virtual bool DoIdleWork() = 0;
	};

	MessagePump() {};
	virtual ~MessagePump() {};

	virtual void Run(Delegate* delegate) = 0;
	virtual void Quit() = 0;
	virtual void ScheduleWork() = 0;
	virtual void ScheduleDelayedWork(const TimeTicks& delay_message_time) = 0;
};
```
```c++
class DefaultMessagePump : public MessagePump
{
public:

	DefaultMessagePump();
	virtual ~DefaultMessagePump() {}

	virtual void Run(Delegate* delegate);
	virtual void Quit();
	virtual void ScheduleWork();
	virtual void ScheduleDelayedWork(const TimeTicks& delay_message_time);

private:
	void Wait();
	void WaitTimeout(const TimeDelta &timeout);
	void Wakeup();

	WaitableEvent event_;
	bool should_quit_;
	TimeTicks delayed_work_time_;

	DISALLOW_COPY_AND_ASSIGN(DefaultMessagePump);
};
```
`MessagePump`将`Run`与`Quit`函数暴露给外部用来控制消息泵开始与停止。同时，当内部没有消息时，会调用`Wait`函数放弃时间片，具体是通过事件机制（`WaitableEvent`）实现
# 运行
```c++
void DefaultMessagePump::Run(Delegate* delegate)
{
	// Quit must have been called outside of Run!
	assert(should_quit_ == false);

	for (;;)
	{
		// 任务态
		bool did_work = delegate->DoWork();
		if (should_quit_)
			break;

		did_work |= delegate->DoDelayedWork(&delayed_work_time_);
		if (should_quit_)
			break;

		if (did_work)
			continue;

		// 空闲态
		did_work = delegate->DoIdleWork();
		if (should_quit_)
			break;

		if (did_work)
			continue;

		// 睡眠态
		if (delayed_work_time_.is_null())
		{
			Wait();
		}
		else
		{
			TimeDelta delay = delayed_work_time_ - TimeTicks::Now();
			if (delay > TimeDelta())
				WaitTimeout(delay);
			else
			{
				// It looks like delayed_work_time_ indicates a time in the past, so we
				// need to call DoDelayedWork now.
				delayed_work_time_ = TimeTicks();
			}
		}
	}

	should_quit_ = false;
}
```
运行过程中`MessagePump`有三种状态
- 任务态 
	只处理`Delegate`的即时任务（`Delegate::DoWork`）以及延时任务（`Delegate::DoDelayedWork`）
- 空闲态
	当`Delegate`的即时任务以及延时任务都为空时，属于空闲状态，处理闲时任务（`Delegate::DoIdleWork`）
- 睡眠态
	当闲时任务也为空时，调用`MessagePump::Wait`相关函数，放弃时间片

# 延时与睡眠