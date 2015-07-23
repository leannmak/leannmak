## 如何使用Samba实现Linux和Windows互访

by leannmak  2015-7-24

### 1. 在Linux上安装Samba:

* 检查Linux上是否已安装Samba：
```bash
$ sudo rpm -qa samba
samba-3.6.23-14.el6_6.x86_64
```
 出现以上结果表明Samba已安装。

* 若查询结果为空，则使用 `yum` 安装：
```
$ sudo yum install samba samba-client
```

* 使用`vim`修改Samba配置:
```
$ sudo vim /etc/samba/smb.conf
```

1) 修改 `security=user` 为 `security=share`

2) 在文件末尾添加以下内容：
```
[share]
comment = leannmak's Files Share
path = /home/leannmak/share
browseable = yes
guest ok = yes
writable = yes
```
`path` 将成为Linux和Windows的共享目录。

* 在Linux中创建 `path` 目录，请确保该目录存在：
```
$ cd /home/leannmak
$ mkdir share
```

* 修改共享目录权限，确保Windows环境下能操作该目录下的文件：
```
$ sudo chmod -R 777 share
```
`-R` 表示对该目录下的所有子目录递归执行该操作。

* 修改完后，查看Samba状态：
```
$ sudo service smb status
smbd (pid  19087) is running...
$ sudo service smb restart
```
若Samba未启动，执行重启，否则直接`start`。


### 2. 在Windows上设置网络驱动器:

* 在 **开始** 菜单中搜索 **运行**，访问 `\\Samba主机的IP地址\share` 。

* 若访问不成功，请执行以下操作：

1) 确保Windows的防火墙已关闭：
控制面板 --> 小图标 --> 管理工具 --> 服务 --> Windows Firewall， 关闭。

2) 检查Samba主机的`selinux`、`iptables`:
* 修改**selinux**配置文件：
```
$ sudo vi /etc/sysconfig/selinux
```
修改SELINUX=enforcing(默认值)为SELINUX=disabled，退出后执行：
```
$ sudo setenforce 0
```
* 关闭Linux防火墙：
```
$ sudo service iptables stop
```

* 打开Windows 计算机 --> 映射网络驱动器，将 `\\samba主机的IP地址\share` 映射为Z:盘。


### 3. 在Windows Z:盘中新建一个`leannmak.txt`文件，在Samba主机中查看：
```
$ sudo cd /home/leannmak/share
$ sudo ll
```
此时可以看到新建的文件信息，Done！

### 4. 系统reboot后，连接出错不能互访怎么办？

* Pls Make Sure that your **Firewalls** on both Windows & Linux have already **closed** !!! 
  Also **SELINUX** is **disabled** !!!
  

  
参考文献: http://my.oschina.net/daisheng/blog/466520
