---
title: Windows动态链接库（dll）浅析 - 3
tags: [Windows, C++]
---

* [Windows动态链接库（dll）浅析 - 1](/2018/04/17/Windows动态链接库（dll）浅析%20-%201/)
* [Windows动态链接库（dll）浅析 - 2](/2018/04/17/Windows动态链接库（dll）浅析%20-%202/)

## 6. DLL的深入使用

### 6.1 DllMain函数

　　Windows在加载DLL的时候，需要一个入口函数，就如同控制台或DOS程序需要main函数、WIN32程序需要WinMain函数一样。在前面的例子中，DLL并没有提供DllMain函数，应用工程也能成功引用DLL，这是因为Windows在找不到DllMain的时候，系统会从其它运行库中引入一个不做任何操作的缺省DllMain函数版本，并不意味着DLL可以放弃DllMain函数。

　　根据编写规范，Windows必须查找并执行DLL里的DllMain函数作为加载DLL的依据，它使得DLL得以保留在内存里。这个函数并不属于导出函数，而是DLL的内部函数。这意味着不能直接在应用工程中引用DllMain函数，DllMain是自动被调用的。

　　我们来看一个DllMain函数的例子：

    BOOL APIENTRY DllMain(HANDLE hModule,DWORD ul_reason_for_call, LPVOID lpReserved)
    {
             switch(ul_reason_for_call)
             {
             case DLL_PROCESS_ATTACH:
                       printf("process attach of dll\n ");
                       break;
             case DLL_THREAD_ATTACH:
                       printf("thread attach of dll\n ");
                       break;
             case DLL_THREAD_DETACH:
                       printf("thread detach of dll\n ");
                       break;
             case DLL_PROCESS_DETACH:
                       printf("process detach of dll\n ");
                       break;
             }
             
             return TRUE;
    }

　　DllMain函数在DLL被加载和卸载时被调用，在单个线程启动和终止时，DLLMain函数也被调用，ul_reason_for_call指明了被调用的原因。原因共有4种，即PROCESS_ATTACH、PROCESS_DETACH、THREAD_ATTACH和THREAD_DETACH。

　　修改上一节中的DllAdd项目中的dllmain.cpp代码如下：

    #include "stdafx.h"
    #include <stdio.h>
    
    BOOL APIENTRY DllMain( HMODULE hModule,
                           DWORD  ul_reason_for_call,
                           LPVOID lpReserved
                         )
    {
        switch (ul_reason_for_call)
        {
        case DLL_PROCESS_ATTACH:
    		printf("process attach of dll\n ");
    		break;
    	case DLL_THREAD_ATTACH:
    		printf("thread attach of dll\n ");
    		break;
    	case DLL_THREAD_DETACH:
    		printf("thread detach of dll\n ");
    		break;
    	case DLL_PROCESS_DETACH:
    		printf("process detach of dll\n ");
    		break;
        }
        return TRUE;
    }
　　
    然后重新生成项目。这时再次执行DllCallDynamic项目，会看到输出结果如下：

    process attach of dll
    result:100
    process detach of dll    

　　这一输出顺序验证了DllMain被调用的时机。
　　
### 6.2 __stdcall函数

　　如果通过VC++编写的DLL欲被其他语言编写的程序调用，应将函数的调用方式声明为__stdcall方式，WINAPI都采用这种方式，而C/C++缺省的调用方式却为__cdecl。__stdcall方式与__cdecl对函数名最终生成符号的方式不同。若采用C编译方式(在C++中需将函数声明为extern"C")，__stdcall调用约定在输出函数名前面加下划线，后面加“@”符号和参数的字节数，形如_functionname@number；而__cdecl调用约定仅在输出函数名前面加下划线，形如_functionname。Windows编程中常见的几种函数类型声明宏都是与__stdcall和__cdecl有关的（节选自windef.h）：

    #define CALLBACK __stdcall //这就是传说中的回调函数
    #define WINAPI __stdcall   //这就是传说中的WINAPI
    #define WINAPIV __cdecl
    #define APIENTRY WINAPI  //DllMain的入口就在这里
    #define APIPRIVATE __stdcall
    #define PASCAL __stdcall

　　在lib.h 中，应这样声明add函数：

    int __stdcall add(int x,int y);

　　在应用工程中函数指针类型应定义为：

    typedef int(__stdcall *lpAddFun)(int,int);

　　若在lib.h 中将函数声明为__stdcall调用，而应用工程中仍使用typedef int (*lpAddFun)(int,int)，运行时将发生错误（因为类型不匹配，在应用工程中仍然是缺省的__cdecl调用），弹出如图所示的异常。
![](https://img-blog.csdnimg.cn/img_convert/b618dc6021ade4fe5d34b88a432eae61.png)　　
　　
## 7. DLL的应用

　　动态链接库（dll）实现了库的共享，体现了代码的重用思想，应用范围非常广泛：

 * 通用算法
 * 纯资源dll
 * 通信控制dll
 * Windows模块
　　Windows控制面板、ODBC驱动程序、ActiveX控件、COM都是使用dll编程

## 8. DLL木马

### 8.1 原理

　　DLL木马的实现原理是编程者在DLL中包含木马程序代码，随后在目标主机中选择特定目标进程，以某种方式强行指定该进程调用包含木马程序的DLL，最终达到侵袭目标系统的目的。
　　
　　正是DLL程序自身的特点决定了以这种形式加载木马不仅可行，而且具有良好的隐藏性：

　　（1）DLL程序被映射到宿主进程的地址空间中，它能够共享宿主进程的资源，并根据宿主进程在目标主机的级别非法访问相应的系统资源；
　　（2）DLL程序没有独立的进程地址空间，从而可以避免在目标主机中留下“蛛丝马迹”，达到隐蔽自身的目的。
　　
　　DLL木马注入其它进程的方法为远程线程插入，是通过在另一个进程中创建远程线程的方法进入那个进程的内存地址空间。将木马程序以DLL的形式实现后，需要使用插入到目标进程中的远程线程将该木马DLL插入到目标进程的地址空间，即利用该线程通过调用WindowsAPI“LoadLibrary”函数来加载木马DLL，从而实现木马对系统的侵害。
　　
### 8.2 DLL木马注入程序
　　(待完成)


##[参考文档]
1. http://blog.csdn.net/inrgihc/article/details/49283193
2. http://blog.csdn.net/heyabo/article/details/8721611
3. http://www.slsup.com/post/54.html

