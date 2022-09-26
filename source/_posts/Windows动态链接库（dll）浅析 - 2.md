---
title: Windows动态链接库（dll）浅析 - 2
date: 2018-05-14 17:28
tags: [Windows, C++]
---

* [Windows动态链接库（dll）浅析 - 1](/2018/04/17/Windows动态链接库（dll）浅析%20-%201/)
* [Windows动态链接库（dll）浅析 - 3](/2018/04/17/Windows动态链接库（dll）浅析%20-%203/)

## 5. DLL的编写

### 5.1 一个简单的dll项目

　　上面用静态链接库的方式提供了add函数接口，下来我们看看用动态链接库的方式如何实现同样的功能。
　　首先，在VS中创建一个“动态链接库”项目DllAdd。
![创建“动态链接库”项目DllAdd](https://img-blog.csdnimg.cn/img_convert/72503fb9910d0938f18e06ee81066d88.png)
　　然后，为项目添加lib.cpp及lib.h文件，源码如下：
#### 　　lib.cpp:

    #include "lib.h"

    int add(int x, int y)
    {
    	return x + y;
    }
    
#### 　　lib.h:

    #ifndef __LIB_H__
    #define __LIB_H__
    
    #ifdef DLLADD_EXPORTS
    
    extern "C" _declspec(dllexport)
    int add(int x, int y);
    
    #else
    
    extern "C" _declspec(dllimport)
    int add(int x, int y);
    
    #endif
    
    #endif

　　最后，在项目属性页中“C/C++ -> 预处理器 -> 预处理器定义”中增加“DLLADD_EXPORTS”。编译生成项目，可以得到DllAdd.lib。
![](https://img-blog.csdnimg.cn/img_convert/2d10a1b14271d6d8cae14aba4e238cae.png)
　　
　　分析上述代码，DllAdd工程中的lib.cpp 文件与前一节静态链接库版本完全相同，不同在于lib.h 对函数add的声明前面添加了LIB_API宏的定义。当“DLLADD_EXPORTS”这个宏有定义时，这个语句的含义是声明函数add为DLL的“extern"C" __declspec(dllexport)”导出函数，否则为”extern"C" __declspec(dllimport)”导入函数。我们通过项目属性页为DllAdd工程增加预处理器DLLADD_EXPORTS，这样这个工程内时，add就被定义成导出函数了，当lib.h文件给调用者使用时，由于调用者的工程中没有该宏的定义，所以它的add函数就被定义成了导入函数。
　　
### 5.2 dll导出函数

　　dll中导出函数的声明方式有两种：一种是像上一个小节给出的示例一样，在函数声明上加上“_declspec(dllexport)”；另一种方法是采用“模块定义(.def)”文件声明，“.def”文件为链接器提供了有关被链接程序的导出、属性等方面的信息。
　　
　　下面给出一个.def文件示例

    ; lib.def : 导出DLL函数
    LIBRARY DllAdd
    EXPORTS
    add @ 1
    
 * 说明：
  * LIBRARY语句说明.def文件相应的DLL；
  * EXPORTS语句后列出要导出函数的名称；
  * 在.def文件中的导出函数名后加@n，表示要导出函数的序号为n（在后面进行显示函数调用时，这个序号将发挥其作用）；

### 5.3 dll的调用方式

　　动态链接库的调用方法包括两种：隐式调用和显示调用。

#### 5.3.1 隐式调用（静态调用）

　　隐式调用也称为静态调用，是由编译系统完成对dll的加载和应用程序结束时卸载。当调用某个DLL的程序结束时，若系统中还有其他程序使用该dll，则Windows对dll的应用记录减1，直到所有使用该dll的程序都结束时才释放它。静态调用方式同静态链接库的调用方式相同，特点也是简单实用，但不如动态调用方式灵活。
　　下面我们新建一个Windows控制台程序DllCallStatic，并修改DllCallStatic.cpp如下：

    #include "stdafx.h"
    #include <stdio.h>
    
    #include "../../DllAdd/DllAdd/lib.h"
    
    #pragma comment(lib, "../../DllAdd/Debug/DllAdd.lib")
    
    int main(int argc, char*argv[])
    {
    	int result = add(2048, 52);
    
    	printf("%d\n", result);
    
    	return 0;
    }

　　注意：
　　（1）将DllAdd.dll复制到DllCallStatic项目的工作目录；
　　（2）在DllCallStatic工程中没有对DLLADD_EXPORTS宏的定义，故add在lib.h头文件中已经被定义称为了extern"C" __declspec(dllimport)导入函数。

    编译并运行，查看结果。
　　
#### 5.3.2 显式调用（动态调用）
　　
　　显式调用是指使用由“LoadLibrary - GetProcAddress - FreeLibrary”系统API提供的三位一体“DLL加载 - DLL函数地址获取 - DLL释放”方式，这种调用方式也被称为DLL的动态调用。动态调用方式的特点是完全由编程者用API函数加载和卸载DLL，程序员可以决定DLL文件何时加载或不加载，显式链接在运行时决定加载哪个DLL文件。
　　
　　新建一个Windows控制台程序DllCallDynamic，并修改DllCallDynamic.cpp如下：

    #include "stdafx.h"
    #include<stdio.h>
    #include<windows.h>
    
    typedef int(*lpAddFun)(int, int); //宏定义函数指针类型
    
    int main(int argc, char*argv[])
    {
    	HINSTANCE hDll;//DLL句柄
    	lpAddFun addFun;//函数指针
    
    	hDll = LoadLibrary(L"..//..//DllAdd//Debug//DllAdd.dll");
    
    	if (hDll != NULL)
    	{
    		addFun = (lpAddFun)GetProcAddress(hDll, "add");
    
    		if (addFun != NULL)
    		{
    			int result = addFun(72, 28);
    			printf("result:%d\n", result);
    		}
    
    		FreeLibrary(hDll);
    	}
    
    	return 0;
    }
　　
　　编译并运行，查看结果。
　　
#### 5.3.3 dll查找顺序
　　为了使需要动态链接库的应用程序可以运行，需要将DLL文件放在操作系统能够找到的地方。Windows操作系统查找DLL的目录顺序为：

1.  **应用程序的可执行文件（*.exe）所在的目录**

2.  **进程的当前工作目录**

3.  **Windows系统目录**：Windows操作系统安装目录的系统子目录，如C:\Windows\ System32。可用GetSystemDirectory函数检索此目录的路径。

4.  **Windows目录**：Windows操作系统安装目录，如C:\Windows\。可用GetWindowsDirectory函数检索此目录的路径。

5.  **搜索目录**：Path环境变量中所包含的自动搜索路径目录，一般包含C:\Windows\和C:\Windows\System32\等目录。可在命令行用Path命令来查看和设置，也可以通过（在“我的电脑”右键菜单中选“属性”菜单项）“系统属性”中的环境变量，来查看或编辑“Path”系统变量和“Path”用户变量。
　　
