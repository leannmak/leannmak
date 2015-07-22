---------------- Docker的官方网站：https://www.docker.com/ ------------------

# Docker学习笔记

by leannmak 2015-7-22

## 1. Docker安装（在Centos 6.5上安装docker）：
Enable EPEL Repo on CentOS
```bash
$ rpm -Uvh http://download.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
```
安装 docker
```
$ yum install docker-io --enablerepo=epel
```
尝试启动 docker daemon 进程
```
$ sudo docker -d &
```
下载 ubuntu 镜像
```
$ sudo docker pull ubuntu:12.04
```
使用 docker 打印 hello world
```
$ sudo docker run ubuntu:12.04 /bin/echo hello world
hello world
```
安装成功！

可通过 exit 命令或 Ctrl+d 来退出终端时，终止所创建的容器。