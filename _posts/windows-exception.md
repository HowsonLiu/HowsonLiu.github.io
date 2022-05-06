---
title: Windows中的异常机制
date: 2020-12-19 14:45:45
categories: windows
tags:
    - windows
description: 主要记录Windows下异常处理的流程  
---
# 概念
- 中断（Interrupt）  
    由外部硬件设备或异步事件产生的
- 异常（Exception）  
    - 故障  
    - 陷阱  
        故障与陷阱都是可恢复的
    - 终止  
        不可恢复的

## IDT（Interrut Decription Table）
- IDT表是位于物理内存中的线性表。当有中断或者异常发生时，CPU会通过IDT来寻找处理函数。  
  
- 操作系统在启动阶段会初始化这个表，系统中每一个CPU都有一份IDT的拷贝  

- IDT的位置与长度由CPU的IDTR寄存器描述。高32位是IDT的基址，低16位是长度  

- IDT有256项，每一项是一个门结构，包括三种门描述符：
    - 任务门，用于CPU任务切换
    - 中断门，描述中断处理程序入口
    - 陷阱门，描述异常处理程序入口


# 数据结构
## CONTEXT
记录环境
```c++
typedef struct DECLSPEC_NOINITALL _CONTEXT {

    //
    // The flags values within this flag control the contents of
    // a CONTEXT record.
    //
    // If the context record is used as an input parameter, then
    // for each portion of the context record controlled by a flag
    // whose value is set, it is assumed that that portion of the
    // context record contains valid context. If the context record
    // is being used to modify a threads context, then only that
    // portion of the threads context will be modified.
    //
    // If the context record is used as an IN OUT parameter to capture
    // the context of a thread, then only those portions of the thread's
    // context corresponding to set flags will be returned.
    //
    // The context record is never used as an OUT only parameter.
    //

    DWORD ContextFlags;

    //
    // This section is specified/returned if CONTEXT_DEBUG_REGISTERS is
    // set in ContextFlags.  Note that CONTEXT_DEBUG_REGISTERS is NOT
    // included in CONTEXT_FULL.
    //

    DWORD   Dr0;
    DWORD   Dr1;
    DWORD   Dr2;
    DWORD   Dr3;
    DWORD   Dr6;
    DWORD   Dr7;

    //
    // This section is specified/returned if the
    // ContextFlags word contians the flag CONTEXT_FLOATING_POINT.
    //

    FLOATING_SAVE_AREA FloatSave;

    //
    // This section is specified/returned if the
    // ContextFlags word contians the flag CONTEXT_SEGMENTS.
    //

    DWORD   SegGs;
    DWORD   SegFs;
    DWORD   SegEs;
    DWORD   SegDs;

    //
    // This section is specified/returned if the
    // ContextFlags word contians the flag CONTEXT_INTEGER.
    //

    DWORD   Edi;
    DWORD   Esi;
    DWORD   Ebx;
    DWORD   Edx;
    DWORD   Ecx;
    DWORD   Eax;

    //
    // This section is specified/returned if the
    // ContextFlags word contians the flag CONTEXT_CONTROL.
    //

    DWORD   Ebp;
    DWORD   Eip;
    DWORD   SegCs;              // MUST BE SANITIZED
    DWORD   EFlags;             // MUST BE SANITIZED
    DWORD   Esp;
    DWORD   SegSs;

    //
    // This section is specified/returned if the ContextFlags word
    // contains the flag CONTEXT_EXTENDED_REGISTERS.
    // The format and contexts are processor specific
    //

    BYTE    ExtendedRegisters[MAXIMUM_SUPPORTED_EXTENSION];

} CONTEXT;
```
## EXCEPTION_RECORD
记录异常信息
```c++
typedef struct _EXCEPTION_RECORD {
    DWORD    ExceptionCode;
    DWORD ExceptionFlags;
    struct _EXCEPTION_RECORD *ExceptionRecord;
    PVOID ExceptionAddress;
    DWORD NumberParameters;
    ULONG_PTR ExceptionInformation[EXCEPTION_MAXIMUM_PARAMETERS];
    } EXCEPTION_RECORD;

typedef EXCEPTION_RECORD *PEXCEPTION_RECORD;
```

# 流程
![1](process.png)  