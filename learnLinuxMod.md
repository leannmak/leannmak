## Linux权限管理

    by leannmak    2015-7-27

### 4 roles:

```bash
$ ll
total 1
drwxr-xrwx.  3 root root 4096 Jul 27 13:35 leannmak
```
* user (u)
* group (g)
* others (o)
* root： 超级管理员。

### 文件权限：

* r (read): 可读取此一文件的实际内容，如读取文本文件的文字内容等；
* w (write): 可以编辑、新增或者是修改该文件的内容(但不含删除该文件)；
* x (execute): 该文件具有可以被系统执行的权限。

### 目录权限：

**r (read contents in directory)**：

* 表示具有读取目录结构列表的权限，所以当你具有读取(r)一个目录的权限时，表示你可以查询该目录下的文件名数据。
* 如可以利用 ls 这个指令将该目录的内容列表显示出来。

**w (modify contents of directory)**：

表示你具有异动该目录结构列表的权限，也就是底下这些权限：

* 建立新的文件与目录；
* 删除已经存在的文件与目录(不论该文件的权限为何！)；
* 将已存在的文件或目录进行更名；
* 搬移该目录内的文件、目录位置。

总之，目录的w权限主要与该目录底下的文件名异动有关。

**x (access directory)**：

* 目录不可以被执行，目录的x代表的是用户能否进入该目录成为工作目录的用途。
* 所谓的工作目录(work directory)就是你目前所在的目录。
* 变换目录的指令是 `cd` (change directory)。

### 修改文件权限、拥有者、群组：

```bash
$ chmod [[u/g/o][+/-][r/w/x]] [filename] (-R)
$ chown
$ chgrp
```
也可用数字操作权限，其中r，w，x分别为4，2，1。

### 绝对路径与相对路径：

* 绝对路径：由根目录(/)开始写起的文件名或目录名称， 例如 `/home/dmtsai/.bashrc` ；
* 相对路径：相对于目前路径的文件名写法，例如 `./home/dmtsai` ，开头不是 / 就属于相对路径的写法。

* `.` ：代表当前的目录，也可以使用 ./ 来表示；
* `..` ：代表上一层目录，也可以 ../ 来代表；
* 如果档名之前多一个 `.` ，则代表这个文件为隐藏文件。

参考文献：http://vbird.dic.ksu.edu.tw/linux_basic/0210filepermission.php

