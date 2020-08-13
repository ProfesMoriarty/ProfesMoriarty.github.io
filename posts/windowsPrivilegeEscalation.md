---

layout: default

---



# Windows权限提升

本篇文章通篇源自于网上前人的提炼总结，为个人复现学习记录使用。





## 甜土豆 SweetPotato

[payload](https://github.com/uknowsec/SweetPotato)

测试结果：

* Win10: defender直接杀
* Win8:未被杀，执行报错 Cannot perform NTLM interception, neccessary priveleges missing.  Are you running under a Service account?
* Win7:未被杀，执行报错 Cannot perform NTLM interception, neccessary priveleges missing.  Are you running under a Service account?
* XP：未被杀，缺少.net环境
* Server2019:defender直接杀
* Server2016:defender直接杀
* Server2012:未被杀，缺少.net环境
* Server2008:未被杀，执行成功
* Server2003:未被杀，缺少.net环境