---
title: Windows SEH & VEH
date: 2020-12-20 16:33:49
categories: windows
tags:
    - windows
description: 主要介绍Windows下异常处理中的SEH以及VEH
---
# SEH
SEH（Structured Exception Handling），结构化异常处理，是Windows用于自身除错的一种表示。用户模式以及内核模式均能使用。作用范围仅限于当前线程
## 存放位置
在TEB（Thread Environment Block，线程环境块）的头部偏移0处，存放着TIB（Thread Information Block，线程信息块）。在TIB的头部偏移0处，存放着异常处理链表  
  
在x86的用户模式下，fs段寄存器指向当前线程的TEB数据。因此通过fs:[0]可以获取到异常处理链表头  

![](2.jpg)  

链表的尾节点Next为-1，Handler是系统设置的一个终结处理函数

由于TEB是线程的私有数据，因此SEH机制的作用范围仅限于当前线程  
## 安装与卸载
由于fs:[0]一直指向链表头，因此安装只需要新增一个链表节点（`EXCEPTION_REGISTRATION_RECORD`，简称ERR），然后令fs:[0]指向新节点，新节点的next指向旧链表头即可  
  
卸载同理。异常处理链的添加与删除都只能在链表头部执行。  

根据SEH的设计要求，他的作用范围与安装他的函数相同，所以通常在函数头部安装SEH异常处理程序，在函数返回前卸载

```asm
assume fs:nothing       // 编译器规定
push offset SEHandler   
push fs:[0]             // 这两个push在栈上构造一个EXCEPTION_REGISTRATION_RECORD结构
mov fs:[0], esp         // 插入至链表头
```

```asm
mov esp, dword ptr fs:[0]   // 卸载
pop dword ptr fs:[0]        // 平衡
```

因此SEH是基于栈帧的（所以大家才说要在main函数头部就要安装异常处理程序啊）
## 异常分发与线程处理
在无调试器的情况下，流程如下：
1. 调用VEH ExceptionHandler，若返回继续执行，则直接返回，否则进行SEH部分处理
2. SEH部分处理，也叫线程处理。遍历当前线程的异常处理链表，根据不同返回值进行不同的处理
    - `ExceptionContinueExecution`：已解决，回去重新执行。修改的信息通过`CONTEXT`结构传递
    - `ExceptionContinueSearch`：未解决，找下一个试试
    - `ExceptionNestedException`：帮你时我也崩了，也叫嵌套异常
    - `ExceptionCollidedUnwind`：栈展开时再次发生了异常
3. 调用VEH ContinueHandler处理  

## 异常处理的栈展开（stack unwinding）
发生异常后，如果当前函数并不能处理此异常，那么他会向上查找异常处理。途中编译器会保证局部对象都会被销毁，这个过程叫做栈展开。
```c++
void func1(){
    Obj a;
    throw 100;
}

void func2() throw (int) {
    Obj* b = new Obj;
    func1();
}

void func3(){
    try {
        func2();
    }
    catch (int i) {
        cout << i << endl;
    }
}
```
在这里，a会正确释放，但b会内存泄漏  

这里就能解释，为什么析构函数不要抛出异常了。
## MSC编译器对异常处理的增强
### 使用方法
关键语句：`__try`，`__except`，`__finally`
```c
__try {
    int* a = NULL;
    *a = 100;
}
__exception(printf("In Filter"), EXCEPTION_EXECUTE_HANDLER) {
    printf("In Handler")
}

__try {

}
__finally {
    printf("In Finally")
}
```
Filter的返回值：
- `EXCEPTION_EXECUTE_HANDLER`：异常在预料之中，接下来执行Handler
- `EXCEPTION_CONTINUE_SEARCH`：不处理异常
- `EXCEPTION_CONTINUE_EXECUTION`：异常被修复，返回现场执行

注意与上面的SEH异常处理返回值不能混用
### 实现
按照设计，感觉每个`__try/__exception`，`__try/__finally`都正好对应一个ERR结构，但是实际上并不是直接这样设计。  

对于单个函数，里面内部嵌套了多个`__try/__xx`块，例如：
```c
void func() {
    __try {
        int* a = NULL;
        __try {
            *a = 100;
        }
        __except(filter1(), EXCEPTION_CONTINUE_SEARCH) {

        }
    }
    __except(filter2(), EXCEPTION_EXECUTE_HANDLER) {
        
    }
}
```
对于这个函数，MSC编译器只会生成一个ERR结构，并插入当前线程的异常处理链表中。ERR结构中的handler，是MSC的一个库函数（VC6.0里其名字为`__exception_handler3`）。  

流程如下：  
1. 准备工作：将`__except`的参数，编译成独立的函数Filter（如果是`__finally`，则Filter为NULL）；块内容，编译成独立的函数Handler。之后存放到`__EH3_EXCEPTION_REGISTRATION::ScopeTable`中。对`__try`块，生成一个`SCOPETABLE_ENTRY`，并按照出现顺序标记Try块索引。简单来说，就是封装`__exception`，标记`__try`。
2. 函数头部布置`CPPEH_RECORD`结构，记录上面封装信息的映射关系，安装SEH异常处理函数`__exception_handler3`
3. 每进入一个try，设置索引值，再执行内容
4. 函数返回前，卸载SEH结构

以上所有操作，都是在编译期插入指令，运行时执行来完成  

对于嵌套函数，编译器会使用多个ERR结构 
## C++中的增强异常处理
与MSC类似，但是有下列不同：
- 无finally
- 提供throw
    内部是调用了`__CxxThrowException` -> `kernel32!RaiseException`，并使用了一个MagicCode作为异常代码区分C++异常与其他异常（不同版本MagicCode不同）
- catch
    调用了`__CxxFrameHandler`，里面有try、catch的位置信息以及异常类型信息，所做工作与`__exception_handler3`十分类似

最后注意，在MSC编译器中，同一个函数不能混用C++异常处理与编译器异常处理。当然子调用可以。
## 顶层异常处理
顶层异常处理是系统设置的一个默认异常处理程序  

![](1.jpg)

Windows在创建进程之后，进程并不是直接从程序入口点开始运行的。在XP中，他的实际启动位置为`kernel32!BaseProcessStartThunk`。在那个函数中，系统会给他安装一个顶层异常处理函数。同理，`CreateThread`创建线程也是，会先跑到`kernel32!BaseThreadStartThunk`中。  

因此，咱们所有创建的进线程，实际上都包含了顶层异常处理函数。
## 异常程序的安全性
由于SEH结构是存储在栈上的，而栈上的数据安全性有时无法得到保证。例如程序接受恶意输入导致溢出攻击时，栈中的SEHandler很容易被覆写为恶意代码。因此微软提供了两种机制保证SEH安全性
### SafeSEH
从.NET 2003开始，编译PE文件时加入了SafeSEH开关。  

在编译阶段提取所有异常处理程序的相对虚拟地址（RVA），放入一个表，存到PE头中

运行时，系统会调用`RtlIsValidHandler`对SEHanlder进行验证。那么通过inline asm设置fs:[0]这种方法设置的SEH就是无效的
### SEHOP
Structured Exception Handling Overwrite Protection。SEH覆写保护机制。在win7及后续版本加入。

主要检测两点：
- 检测SEH链完整性，每个节点必须存于栈中，并且可以正确访问
- 检测最后一个节点的异常函数是否为`ntdll!FinalExceptionHandler`