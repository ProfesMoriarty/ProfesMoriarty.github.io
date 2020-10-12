---


layout: default
---



# Cobalt Strike -- Malleable C2 Profile

## 0x01 Malleable C2 Profile 基本构成部分



CobaltStrike提供Melleable C2 Profile文件可以修改Beacon默认的流量以及行为特征，从而达到隐藏自身的效果。按照官方文档的说法，创建配置文件最好的办法就是修改现有的配置文件，[Github](https://github.com/rsmudge/Malleable-C2-Profifiles)上有一个配示例仓库。可以从该仓库中参考定制Profile文件。



cs自带了profile检查工具：c2init

## 0x02 各部分详解

### http-get

受控机会主动向teamserver发送get上线请求包。

### http-post

teamserver通过post请求向受控机下发命令

### https-certificate

用来设置ssl上线，

### stage

Stage用来控制Beacon

### http-stager

按照cs的官方文档， http-stager用来自定义HTTP staging 过程

### Profile Variants

在同一个配置文件中配置多个配置模板



## 0x03 名词解释

1. Stage
2. Stager
3. Staging
4. Beacon：Beacon就是运行在被控机器上的payload 也就是shellcode







## 参考链接

* [cs官方文档](https://www.cobaltstrike.com/help-malleable-c2)
* [How to Write Malleable C2 Profiles for Cobalt Strike](https://bluescreenofjeff.com/2017-01-24-how-to-write-malleable-c2-profiles-for-cobalt-strike/)
* [深入研究cobalt strike malleable C2配置文件](https://xz.aliyun.com/t/2796)

  

