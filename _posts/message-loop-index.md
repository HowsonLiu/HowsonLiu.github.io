---
title: nbase 消息循环
date: 2020-04-19 15:10:22
tags:
    - chrome
    - duilib
---
消息循环是`nbase`库比较重要的一个部分，也比较庞大，在这里先记录其重要的部分，也算一个索引，后面在详细记录实现部分
`nbase`将消息循环按照功能拆分成及部分
- `MessagePump` 控制消息循环的开始跟停止
- `MessageLoop` 消息循环中的消息队列实现
# MessagePump
消息泵。驱动各种类型的消息循环，简单来说就是实现死循环的类
它主要提供几个方法
- Run 启动消息循环
- Quit 结束消息循环
- ScheduleWork 处理任务
- ScheduleDelayedWork 处理延时任务

`MessagePump`主要充当CPU跟`MessageLoop`间中间人的角色。`MessageLoop`不用操心啥时候有时间片啥时候开始啥时候停止，而`MessagePump`也不用操心要执行什么任务

# MessageLoop
消息循环。主要维护任务队列，提供接口供外界提交任务，提供任务给`MessagePump`执行。
简单的消息循环一定要提供几个方法
- DoWork 处理普通任务
- DoDelayedWork 处理延时任务
- DoIdleWork 处理空闲任务

`nbase`库将任务分成这三种类型