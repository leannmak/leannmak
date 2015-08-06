# Load Balance 学习笔记

by leannmak 2015-8-6

## 1. LVS 与 Keepalived 安装（Centos 6.5）：
### 1.1 下载 LVS 与 Keepalived 软件包
```bash
# 懒得创建新目录，就跟这儿放着了
$ sudo cd /usr/src/

# ipvsadm版本和linux内核是配对好的，我警告你不要坏人姻缘
$ sudo uname -r
2.6.32-504.8.1.el6.x86_64
$ sudo wget http://www.linuxvirtualserver.org/software/kernel-2.6/ipvsadm-1.24.tar.gz

$ sudo wget http://www.keepalived.org/software/keepalived-1.2.1.tar.gz
```
### 1.2 安装 LVS 与 Keepalived 软件包

#### 1.2.1 LVS

--------------------------------- 不想踩雷必看 ---------------------------------

编译LVS之前要先做个目录的 **软链接**，否则会报错，IMPORTANT！

HOW?

一般来说，只要做如下处理就可以了：
```
$ sudo uname -r
2.6.32-504.8.1.el6.x86_64
$ sudo ln -s /usr/src/kernels/2.6.32-504.8.1.el6.x86_64/  /usr/src/linux

# 查看软链接是否已建立
$ sudo ll /usr/src/linux
lrwxrwxrwx. 1 root root 44 Aug  6 10:34 /usr/src/linux -> /usr/src/kernels/2.6.32-504.8.1.el6.x86_64/
```
BUT，蛋疼的是， `ll` 之后你很有可能会发现 `->` 后头内一溜简直闪瞎眼。Why? 当然是因为链过去的目标目录不存在啦。

NOW，不想死就 pls follow me =.=
```
# 首先检查一下kernels目录下是什么鬼
$ sudo ll /usr/src/kernels
drwxr-xr-x. 22 root root 4096 Jun  2 23:11 *2.6.32-504.16.2.el6.x86_64*

# 识做啦？
$ rm -rf /usr/src/linux
$ ln -s /usr/src/kernels/2.6.32-504.16.2.el6.x86_64/  /usr/src/linux

# 最后，再查看下创建的软链接
$ ll /usr/src/linux
lrwxrwxrwx. 1 root root 44 Aug  6 10:34 /usr/src/linux -> /usr/src/kernels/2.6.32-504.16.2.el6.x86_64/
```
Good! Now it's not shining. DONE!

接下来就可以愉快地开始编译安装了！请叫我红领巾！

---------------------------------- 雷区END分界线 --------------------------------

开始安装LVS：
```
$ sudo cd /usr/src/
$ sudo tar zxvf ipvsadm-1.24.tar.gz
$ sudo cd ipvsadm-1.24
$ sudo make && make install

# 查看ipvsadm的位置
$ sudo find / -name ipvsadm
/etc/rc.d/init.d/ipvsadm
/sbin/ipvsadm
/usr/src/ipvsadm-1.24/ipvsadm
```
如果还报错————对不起，上帝也看你不顺眼，请自查下RP。

#### 1.2.2 Keepalived

```
$ sudo cd /usr/src/
$ sudo tar zxvf keepalived-1.2.1.tar.gz
$ sudo cd keepalived-1.2.1
$ sudo ./configure  --prefix=/usr/local/keepalived --with-kernel-dir=/usr/src/kernels/2.6.32-504.16.2.el6.x86_64/
Keepalived configuration
------------------------
Keepalived version       : 1.2.1
Compiler                 : gcc
Compiler flags           : -g -O2
Extra Lib                : -lpopt -lssl -lcrypto 
Use IPVS Framework       : Yes
IPVS sync daemon support : Yes
Use VRRP Framework       : Yes
Use Debug flags          : No

$ sudo make && make install

# 查看keepalived的位置
$ sudo find / -name keepalived
/etc/rc.d/init.d/keepalived
...
/usr/local/keepalived/etc/rc.d/init.d/keepalived
...
/usr/local/keepalived/sbin/keepalived
```
--------------------------------- 踩雷的来聊聊天 ---------------------------------

`./configure` 报错了吧？ `configure: error: Popt libraries is required` 什么鬼？

**缺包** 啊同学！会用 `yum` 吗？
```
$ sudo yum install popt-devel
```
完成后再 `./configure` 一下。再遇到问题请自行解决，不要让PO主怀疑你的智商。

--------------------------------- 雷区END分界线 -----------------------------------

为了方便使用，装完后建议做如下配置，当然 it all depends on u ：
```
$ sudo cp /usr/local/keepalived/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/
$ sudo cp /usr/local/keepalived/etc/sysconfig/keepalived /etc/sysconfig/
$ sudo mkdir /etc/keepalived
$ sudo cp /usr/local/keepalived/etc/keepalived/keepalived.conf /etc/keepalived/
$ sudo cp /usr/local/keepalived/sbin/keepalived /usr/sbin/

$ sudo service keepalived start
Starting keepalived:                                       [  OK  ]
```

Continuing ...
