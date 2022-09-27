---
title: Windows动态链接库（dll）浅析 - 1
date: 2018-05-14 17:25
author: Leon
tags: [Windows, C++]
---

* [Windows动态链接库（dll）浅析 - 2](/2018/04/17/Windows动态链接库（dll）浅析%20-%202/)
* [Windows动态链接库（dll）浅析 - 3](/2018/04/17/Windows动态链接库（dll）浅析%20-%203/)

本文档源码下载：[点击这里](/assets/20180417101050329-LibraryTest.zip)

##  1. 链接库的概念

　　动态链接库（Dynamic link Library, dll），是包含了可由多个程序同时使用的代码（函数、类）和数据的“库”。
　　
##  2. 由来
　　在库的发展史上，经历了“无库 - 静态链接库 - 动态链接库”等阶段。
　　早期，代码被简单地附加到调用它的项目中。如果两个程序同时调用一个子程序，就回出现两份这个子程序的代码；如果采用静态链接库，则lib中的指令都被直接包含在最终生成的目标文件中；而dll则可以在exe文件执行时被动态地引用和卸载。

[外链图片转存失败,源站可能有防盗链机制,建议将图片保存下来直接上传(img-50IoDCC1-1658212205302)(https://docs.google.com/drawings/d/e/2PACX-1vR_1901vUYj2IHG1ygCW2aP8BmCP53-1ap0rfaGs1lUg3PUxb6Ek1s7LcHc6GBO-ieIyT_hsMi-Lp5a/pub?w=1235&h=1529)]

##  3. DLL的优缺点
　　dll随处可见，Windows目录下的system32文件夹中会看到kernel32.dll、user32.dll和gdi32.dll，windows的大多数API都包含在这些DLL中。kernel32.dll中的函数主要处理内存管理和进程调度；user32.dll中的函数主要控制用户界面；gdi32.dll中的函数则负责图形方面的操作。

* ### 优点：

 
 * dll节省了内存，实现代码重用；
 * dll有助于促进模块化程序开发。

* ### 缺点：

 * DLL Hell：即DLL地狱，指几个应用程序在使用同一个共享的DLL库时发生版本冲突。

## 4.静态链接库
　　虽然静态链接库不是本文的重点，但通过一个静态链接库的例子有助于帮助我们建立“库”的概念。
　　
### 4.1 使用VS2017创建和使用静态库
![使用VS2017建立静态库项目](https://img-blog.csdnimg.cn/img_convert/794d38fe8930a46366eb370cf34ed628.png)
　　如上图，使用VS2017建立一个静态库工程LibAdd，然后添加lib.cpp和lib.h两个文件，源码如下：
    
　　lib.h:
    
    // lib.h
    #ifndef __LIB_H__    
    #define __LIB_H__    
    
    extern "C" int add(const int x, const int y); //声明为C编译、链接方式的外部函数
    
    #endif

　　lib.cpp:
    
    // lib.cpp
    #include "lib.h"

    int add(const int x, const int y)
    {
    	return x + y;
    }

　　点击菜单“生成”->“生成解决方案”，就得到“LibAdd.lib”，这个文件就是一个静态库，它提供了add功能。将lib.h和LibAdd.lib交给用户后，用户就可以直接使用里面的add函数了。
　　
　　下面，我们从用户角度来演示下怎么使用这个库。
　　
　　新建一个名为AppCall的Windows控制台应用程序。

![新建Windows控制台程序](https://img-blog.csdnimg.cn/img_convert/76cbb695c854a1cd1b77adcc11aabbe8.png)
　　
　　项目中的AppCall.cpp中已经包含了main函数，我们修改AppCall.cpp的内容如下：

    #include "stdafx.h"

    #include <stdio.h>
    #include "..\..\LibAdd\LibAdd\lib.h"

    #pragma comment(lib, "..\\..\\LibAdd\\Debug\\LibAdd.lib")

    int main(int argc, char *argv[])
    {
    	int a = 512;
    	int b = 1024;
    
    	int x = add(a, b);
    
    	printf("result:%d.", x);
    
        return 0;
    }

　　编译运行，可以看到程序的执行结果。
　　代码中的“#pragma comment(lib, "..\\..\\LibAdd\\Debug\\LibAdd.lib")” 的意思是指本文件生成的.obj文件应该与“LibAdd.lib”一起链接。如果不适用“#pragma commet”指定，则可以直接在属性页中的“链接器->输入->附加依赖项”中填入库文件的路径。

> #### 附：VS中的包含目录和库目录
　　上面的代码中，“#include "..\..\LibAdd\LibAdd\lib.h"” 这行使用了相对路径来指定所需要包含的头文件路径。但这样的路径太复杂，而且一旦项目位置更改，就需要修改代码。
　　VS中可以通过“项目属性页”中的“包含目录”和“附加包含目录”两个配置选项，增加常用头文件所在路径：
![包含目录](https://img-blog.csdnimg.cn/img_convert/b3febe6a10b66e30640a5b2893e483d9.png)
![附加包含目录](https://img-blog.csdnimg.cn/img_convert/b98401ba19146709a69f946d6246314f.png)
　　同样，“库目录”和“附加库目录”可以用来指定所需要使用的lib文件所在目录。
![](https://img-blog.csdnimg.cn/img_convert/1f2b0daddd17360596ce0a6067185369.png)
![](https://img-blog.csdnimg.cn/img_convert/cf97dcbbdeb16eb92afebca33ef231a5.png)
　　额外的，lib还需要在“项目依赖项”中写入lib文件名
![](https://img-blog.csdnimg.cn/img_convert/49813d8521c95d044161243e792ae5b7.png)

### 4.2 如何调试库项目

　　库项目不能单独执行，我们需要在项目属性页中“调试->命令”中配置调用该库的exe文件的路径，这样就可以对该库项目进行调试了。
　　
　　更好的办法是，将库项目和应用项目（就是调用库的那个VC项目）放置在同一个工作区，只对应用项目进行调试，前提是保证两个项目都生成了正确的调试信息。在应用项目中调用库中函数的语句处设置断点，执行到此处后执行“单步进入”（默认快捷键：F11），这样就能够执行到库中的函数代码。
　　
　　上述办法对静态链接库和动态链接库都适用。
