## 如何配置DNS 

by leannmak 2015-8-3

### 1 架设DNS所需要的软件
  ```bash
  $ sudo yum install bind
  $ sudo yum install bind-chroot
  
  $ sudo rpm -qa | grep '^bind'
  bind-libs-9.8.2-0.30.rc1.el6_6.3.x86_64
  bind-chroot-9.8.2-0.30.rc1.el6_6.3.x86_64
  bind-9.8.2-0.30.rc1.el6_6.3.x86_64
  bind-utils-9.8.2-0.30.rc1.el6_6.3.x86_64
  ```

  安装之前可以先用 `rpm` 查看一下，一定要保证以上四个都有，IMPORTANT！


### 2 DNS基础

**IMPORTANT TIPS**
- DNS：域名服务器，主要用于域名与IP的管理和解析，目前应用最广的是Berkeley Internet Name Domain, BIND。
- DNS利用类似树状目录的架构，将主机名的管理分配在不同层级的DNS服务器，每个上一层的DNS所记录的信息，只有其下一层的主机名。
- 在 `.` (root)下有六大一般最上层域名如 *.com*，*.org*，*.edu*，*.gov*，*.net*，*.mil*，以及国码最上层域名如 *.cn*。
- 完整主机名FQDN：即 **主机名 + 完整域名**，务必在最后加 `.` 表示最上层root DNS，如 *www.google.com.*。
- zone类型：
  + hint： `.` (root)；
  + master： 主DNS，需要手动配置；
  + slave： 相当于master备库，直接从master取得配置。
- master和slave在查询时并未有固定优先级，遵从DNS **先抢先赢** 原则。


#### 2.1 正解：通过主机名查IP

```
$ sudo dig www.google.com
...
;; ANSWER SECTION:
www.google.com.         59      IN      A       216.58.221.132
;; AUTHORITY SECTION:
google.com.             8981    IN      NS      ns1.google.com.
...
```

**正解RR标志说明**

- A/AAAA ：查询IP的记录
```
$ sudo dig www.google.com
主机名.   IN  A           IPv4 的 IP 地址
主机名.   IN  AAAA        IPv6 的 IP 地址
```

- NS ：查询管理领域名(zone)的服务器主机名
```
$ sudo dig -t ns google.com
领域名.   IN  NS          管理这个领域名的服务器主机名字.
```

- SOA ：查询管理领域名的服务器管理信息
```
$ sudo dig -t soa google.com
领域名.  IN  SOA  [Master DNS   Email   Serial   Refresh   Retry   Expire   Minumum TTL]  
```
  (1) Master DNS ：master服务器主机名。
  
  (2) Email ： 管理员邮箱，由于 `@` 字符在zone file里有特殊定义，一般用 `.` 替换。
  
  (3) Serial ： zone file的新旧标志，越大代表越新。 slave DNS将据此判断是否主动从master下载新的zone file。每次修改zone file请务必加大该序列号，此时master重启DNS后会主动告知slave更新。

  (4) Refresh ： slave向master要求数据更新的频率。

  (5) Retry ： slave对master联机失败后，重新尝试联机的时间间隔。

  (6) Expire ： slave对master联机失败时的尝试联机失效时间，超时后slave不再尝试更新，并将删除该次下载的zone file。

  (7) Minumum TTL ： zone file中每笔记录的默认快取时间。


- CNAME ：设定某主机名的别名(alias)
```
$ sudo dig www.google.com
主机别名.   IN  CNAME       实际代表这个主机别名的主机名字.
```

- MX ：查询某领域名的邮件服务器主机名
```
$ sudo dig -t mx google.com
领域名.   IN  MX          number  接收邮件的服务器主机名字
```
其中，number用于设置优先级，数字越小的上层邮件服务器优先收受信件。


#### 2.2 反解：通过IP查主机名

```
$ sudo dig -x 216.239.32.10
...
;; ANSWER SECTION:
10.32.239.216.in-addr.arpa. 86400 IN    PTR     ns1.google.com.
...
```

**反解RR标志说明**

PTR ：查询IP所对应的主机名。反解的zone必须将IP反过来写，并以 `.in-addr.arpa.` 结尾。


### 3 配置cache-only DNS

* cache-only也就是快取DNS，只保留DNS的转发功能forwarding，而不做具体查询。

* 为了系统安全，一般会在防火墙主机上加装一个cache-only的DNS服务。

* 配置时，只需修改Bind的主配置文件 */etc/named.conf* ：

```
$ sudo vim /etc/named.conf
// named.conf
options {
        listen-on port 53 { any; };		//可不设定，代表全部接受
        directory       "/var/named";	//数据库默认目录
        dump-file       "/var/named/data/cache_dump.db";	//统计信息
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { any; };		//可不设定，代表全部接受
        recursion yes;		//将自己视为客户端的一种查询模式
        forward only;		//只转发
        forwarders {		//设置接受转发的上层DNS，IMPORTANT！
        192.168.182.64; 
        192.168.182.16;
        };
};
```

(1) `listen-on port 53 { any; };`

  `port 53`是DNS默认端口，设为 `any` 时表示对整个主机系统的所有网络接口进行监听，可根据需要自行修改IP，默认一般为 `localhost` 。

(2) `directory "/var/named";`

  正、反解的zone file预设目录。由于chroot的关系，最终这些文件会被主动链接到 */var/named/chroot/var/named/* 目录。

(3) `dump-file, statistics-file, memstatistics-file`

  与named服务有关的统计信息记录文档，可不设置。

(4) `allow-query { any; };`

  设定有权对本机DNS服务提出查询请求的客户端，需要同时开放防火墙。预设只对 `localhost` 开放。

(5) `forward only ;`

  表示当前DNS服务器仅进行forwarding，将查询权交给上层DNS，即使存在 `.` 的zone file资料也不会使用，是cache only DNS的最常见设定。

(6) `forwarders { 192.168.182.64;  192.168.182.16; } ;`

  设定接受forwarding的上层DNS，考虑到上层DNS可能会挂点，因此设定时建议添加多部上层DNS，这些DNS一般为主从（master/slave）关系。


* 配置完成后，启动或重新启动named服务，查看：

```
$ sudo service named start
$ sudo chkconfig named on

$ sudo netstat -utlnp | grep named
tcp        0      0 192.168.182.14:53           0.0.0.0:*                   LISTEN      19141/named         
tcp        0      0 10.0.100.14:53              0.0.0.0:*                   LISTEN      19141/named         
tcp        0      0 127.0.0.1:53                0.0.0.0:*                   LISTEN      19141/named         
tcp        0      0 127.0.0.1:953               0.0.0.0:*                   LISTEN      19141/named         
tcp        0      0 ::1:953                     :::*                        LISTEN      19141/named         
udp        0      0 192.168.122.1:53            0.0.0.0:*                               19141/named         
udp        0      0 192.168.182.14:53           0.0.0.0:*                               19141/named         
udp        0      0 10.0.100.14:53              0.0.0.0:*                               19141/named         
udp        0      0 127.0.0.1:53                0.0.0.0:*                               19141/named        

$ sudo tail -n 30 /var/log/messages | grep named    
Aug  4 09:56:03 ceilometer1 named[19141]: starting BIND 9.8.2rc1-RedHat-9.8.2-0.30.rc1.el6_6.3 -u named -t /var/named/chroot
...
Aug  4 09:56:03 ceilometer1 named[19141]: loading configuration from '/etc/named.conf'
...
Aug  4 09:56:03 ceilometer1 named[19141]: running
```

(1) DNS默认会同时启用UDP/TCP的 `port 53` 。

(2) 每次重新启动DNS后，务必检查 */var/log/messages* ，出现以上语句表示 */var/named/etc/named.conf* 加载成功。 


### 4 配置master/slave以及子域DNS

#### 4.1 架构规划：
+ **master DNS** : 
  - eth0 ：10.0.100.64	(对外)
  - eth1 ：192.168.182.64	(对内)
  - 主机名与RR标志 ：
    * master.centos.leannmak (NS, A)
    * www.centos.leannmak (A)
    * linux.centos.leannmak (CNAME)
    * ftp.centos.leannmak (CNAME)
    * forum.centos.leannmak (CNAME)
    * www.centos.leannmak (MX)
+ **slave DNS** ：
  - IP ：192.168.182.16
  - 主机名与RR标志 ：
    * slave.centos.leannmak (NS, A)
    * clientlinux.centos.leannmak(A)
+ **子域DNS** ： 
  - IP ：192.168.182.15
  - 主机名与RR标志 ：
    * master.wiki.centos.leannmak (NS, A)
    * www.wiki.centos.leannmak (A)
    * linux.wiki.centos.leannmak (CNAME)
    * ftp.wiki.centos.leannmak (CNAME)
    * forum.wiki.centos.leannmak (CNAME)
    * www.wiki.centos.leannmak (MX)


#### 4.2 master DNS：

重点需要配置三个文件：
- `named.conf`    #主配置文件
- `named.centos.leannmak`    #自定义域名 *centos.leannmak* 的正解zone file
- `named.192.168.182`   #域名 *centos.leannmak* 对应网段 *192.168.182.0/24* 的反解zone file


##### 4.2.1 主配置 *named.conf*
```
$ sudo vim /etc/named.conf 
 1 // named.config
 2 
 3 options {
 4         directory       "/var/named";
 5         dump-file       "/var/named/data/cache_dump.db";
 6         statistics-file "/var/named/data/named_stats.txt";
 7         memstatistics-file "/var/named/data/named_mem_stats.txt";
 8         allow-query     { any; };
 9         recursion yes;  
10         
11         allow-transfer  { 192.168.182.16; };    //允许IP为 192.168.182.16 的slave进行zone转移
12 };      
13 
14 acl intranet { 192.168.182.0/24; };    //代表 192.168.182.0/24 IP来源（内网）
15 acl internet {! 192.168.182.0/24; any;  };    //代表非 192.168.182.0/24 IP来源（外网）
16 
17 view "lan" {
18         match-clients { "intranet"; };    //吻合 intranet 来源的使用底下的zone
19         
20         zone "." IN {
21                 type hint;
22                 file "named.ca";
23         };      
24         //正解zone file
25         zone "centos.leannmak" IN {
26                 type master;
27                 file "named.centos.leannmak";
28                 allow-transfer  { 192.168.182.16; };
29         };      
30         //反解zone file
31         zone "182.168.192.in-addr.arpa" IN {
32                 type master;
33                 file "named.192.168.182";
34                 allow-transfer  { 192.168.182.16; };
35         };      
36 };      
37 
38 view "wan" {
39         match-clients { "internet"; };    //吻合 internet 来源使用底下的zone
40         
41         zone "." IN {
42                 type hint;
43                 file "named.ca";
44         };     
45
46         zone "centos.leannmak" IN {
47                 type master;
48                 file "named.centos.leannmak.inter";
49         };
50         
51         // 由于外网未用到内网的IP，因此IP反解部分可以忽略
52 };
```

- 第14、15、17、18、36、38-52为DNS的view功能配置，当每台主机上有两张网卡时，可通过view的设置使内外网用户查询得到的IP分离。不需要时直接将这几行内容注释掉即可。

- 若在此配置了view，请务必同时完成 *named.centos.leannmak.inter* 的配置，详见4.2.4.3。 


##### 4.2.2 正解 *named.centos.leannmak*

```
$ sudo vim /var/named/named.centos.leannmak
$TTL 600
# 每次更新该文档时请务必加大序列号，此处为2015073101
@       IN      SOA     master.centos.leannmak. leannmak.www.centos.leannmak.(
          2015073101 3H 15M 1W 1D)

# 针对master的NS和A设定
@       IN      NS      master.centos.leannmak.
master.centos.leannmak. IN      A       192.168.182.64

# 针对slave的NS和A设定
@       IN      NS      slave.centos.leannmak.
slave.centos.leannmak.  IN      A       192.168.182.16
@       IN      MX      10      www.centos.leannmak.

# 针对master的所有相关正解设定
www.centos.leannmak.    IN      A       192.168.182.64
linux.centos.leannmak.  IN      CNAME   www.centos.leannmak.
ftp.centos.leannmak.    IN      CNAME   www.centos.leannmak.
forum.centos.leannmak.  IN      CNAME   www.centos.leannmak.

# 针对slave的A设定
slave.centos.leannmak.  IN      A       192.168.182.16
clientlinux.centos.leannmak.    IN      A       192.168.182.16

# 针对子域的NS和A设定
wiki.centos.leannmak.   IN      Ns      dns.wiki.centos.leannmak.
dns.wiki.centos.leannmak.       IN      A       192.168.182.15
```

**IMPORTANT TIPS**
- `@` 代表zone， 此处 `@` = *centos.leannmak.* 。 
- 正解zone file中描述主机的两种方式：
    * 使用 **FQDN** ：末尾一定要加 `.` ，否则视为hostname。如 *www.centos.leannmak* ，在此处FQDN将被视为 *www.centos.leannmak.@* = *www.centos.leannmak.centos.leannmak.* ；
    * 使用 **hostname** ：如 *www*，相当于 *www.@* = *www.centos.vbird* 。 
- `;` 或 `#` 为注释符号。
- 所有设定一定要从行首开始，前面不可有空格符，若有则代表延续前一行。


##### 4.2.3 反解 *named.192.168.182*

```
$ sudo vim /var/named/named.192.168.182
$TTL    600
@       IN      SOA     master.centos.leannmak. leannmak.www.centos.leannmak.   (
                        2015073101      3H      15M     1W      1D      )

# 针对master的NS和PTR设定
@       IN      NS      master.centos.leannmak.
64      IN      PTR     master.centos.leannmak.
64      IN      PTR     www.centos.leannmak.

# 针对slave的PTR设定
16      IN      PTR     slave.centos.leannmak.
```

**IMPORTANT TIPS**
- `@` 代表zone，此处 `@` = *182.168.192.in-addr.arpa.*； 
- 反解zone file中一般只用IP的最后一个字段表示IP，如 `64` = *192.168.182.64*；
- 在反解中用到主机名时，务必使用FQDN设定。


##### 4.2.4 其他需要配置的文件

###### 4.2.4.1 防火墙 *iptables*

```
$ sudo vim /etc/sysconfig/iptables
...
# eth0 为外网接口 10.0.100.64
-A INPUT -p UDP -i eth0 --dport  53  --sport 1024:65534 -j ACCEPT
-A INPUT -p TCP -i eth0 --dport  53  --sport 1024:65534 -j ACCEPT
...
```

###### 4.2.4.2 *resolv.conf*

```
$ sudo vim /etc/resolv.conf
nameserver 192.168.182.64  //自己的IP一定要最早出现
nameserver 192.168.163.34
```

###### 4.2.4.3  外网view *named.centos.leannmak.inter*

若 `named.config` 中配置了view功能，则务必同时配置该文档。

```
$ sudo vim /var/named/named.centos.leannmak.inter 
$TTL 600
@       IN      SOA     master.centos.leannmak. leannmak.www.centos.leannmak.(
        2015073101 3H 15M 1W 1D)
@       IN      NS      master.centos.leannmak.
master.centos.leannmak. IN      A       10.0.100.64
@       IN      MX      10      www.centos.leannmak.

www.centos.leannmak.    IN      A       10.0.100.64
linux.centos.leannmak.  IN      CNAME   www.centos.leannmak.
ftp.centos.leannmak.    IN      CNAME   www.centos.leannmak.
forum.centos.leannmak.  IN      CNAME   www.centos.leannmak.
```

##### 4.2.5 启动或重启DNS

配置完成后，记得启动或重新启动named服务，并测试及查看相关信息：

```
$ sudo service named start
$ sudo chkconfig named on

$ sudo tail -n 60 /var/log/messages | grep named    
Aug  4 09:56:03 ceilometer1 named[19141]: starting BIND 9.8.2rc1-RedHat-9.8.2-0.30.rc1.el6_6.3 -u named -t /var/named/chroot
...
Aug  4 09:56:03 ceilometer1 named[19141]: **loading configuration from '/etc/named.conf'** 
...
Aug  4 03:08:34 cloudlab064 named[24337]: zone 182.168.192.in-addr.arpa/IN/lan: loaded serial 2015073101
Aug  4 03:08:34 cloudlab064 named[24337]: zone centos.leannmak/IN/lan: loaded serial 2015073101
Aug  4 03:08:34 cloudlab064 named[24337]: zone centos.leannmak/IN/wan: loaded serial 2015073101
Aug  4 03:08:34 cloudlab064 named[24337]: zone centos.leannmak/IN/lan: sending notifies (serial 2015073101)
Aug  4 03:08:34 cloudlab064 named[24337]: running

# 测正解
$ sudo dig www.centos.leannmak
...
;; ANSWER SECTION:
www.centos.leannmak.    600     IN      A       192.168.182.64
...

# 测CNAME
$ sudo dig ftp.centos.leannmak
...
;; ANSWER SECTION:
ftp.centos.leannmak.    600     IN      CNAME   www.centos.leannmak.
www.centos.leannmak.    600     IN      A       192.168.182.64
...

# 测MX
$ dig -t mx centos.leannmak
...
;; ANSWER SECTION:
centos.leannmak.        600     IN      MX      10 www.centos.leannmak.
...

# 测master/slave反解
$ sudo dig -x 192.168.182.64
...
;; ANSWER SECTION:
64.182.168.192.in-addr.arpa. 600 IN     PTR     www.centos.leannmak.
64.182.168.192.in-addr.arpa. 600 IN     PTR     master.centos.leannmak.
...
$ sudo dig -x 192.168.182.16
...
;; ANSWER SECTION:
16.182.168.192.in-addr.arpa. 600 IN     PTR     slave.centos.leannmak.
...
```

#### 4.3 slave DNS：

只需要重点配置一个文件，当然别忘了 `iptables` 和 `resolv.conf` ：
- `named.conf`    # 主配置文件


##### 4.3.1 主配置 *named.conf*
```
$ sudo vim /etc/named.conf
// named.conf
options {
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { any; };
        recursion yes;
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "centos.leannmak" IN {
        type slave;
        file "slaves/named.centos.leannmak";
        masters { 192.168.182.64; };
};

zone "182.168.192.in-addr.arpa" IN {
        type slave;
        file "slaves/named.192.168.182";
        masters { 192.168.182.64; };
};
```

**IMPORTANT TIPS**
- slave的zone file都是从master取得的，默认放在 */var/named/slaves* 目录下。
- 启动named前务必检查zone file的目录权限是否正确，需要与 `named` 这个用户及群组有关：
```
$ sudo ll -d /var/named/slaves
drwxrwx--- 2 named named 4096 Aug  3 10:24 /var/named/slaves
```


##### 4.3.2 启动或重启DNS

配置完成后，记得启动或重新启动named服务，并测试及查看相关信息：

```
$ sudo service named start
$ sudo chkconfig named on

$ sudo tail -n 60 /var/log/messages | grep named    
Aug  4 09:56:03 ceilometer1 named[19141]: starting BIND 9.8.2rc1-RedHat-9.8.2-0.30.rc1.el6_6.3 -u named -t /var/named/chroot
...
Aug  4 09:56:03 ceilometer1 named[19141]: **loading configuration from '/etc/named.conf'** 
...
Aug  4 03:08:34 cloudlab064 named[24337]: running


# 检查zone file是否从master自动创建成功
$ sudo ll /var/named/slaves
total 8
-rw-r--r-- 1 named named 473 Aug  4 15:28 named.192.168.182
-rw-r--r-- 1 named named 646 Aug  4 15:41 named.centos.leannmak

# 检查A是否正确
$ dig master.centos.leannmak @127.0.0.1
...
;; ANSWER SECTION:
master.centos.leannmak. 600     IN      A       192.168.182.64
...

# 检查PTR是否正确
$ dig -x 192.168.182.64 @127.0.0.1
...
;; ANSWER SECTION:
64.182.168.192.in-addr.arpa. 600 IN     PTR     www.centos.leannmak.
64.182.168.192.in-addr.arpa. 600 IN     PTR     master.centos.leannmak.
...
```


#### 4.4 子域DNS

* 子域DNS事实上也就是下层的master，需要有完整的zone相关设定，重点配置三个文件：
- `named.conf`    # 主配置文件
- `named.wiki.centos.leannmak`    # 子域名 *wiki.centos.leannmak* 的正解zone file
- `named.192.168.182`    # 域名 *wiki.centos.leannmak* 对应网段 *192.168.182.0/24* 的反解zone file

* 具体与4.2配置master的详细内容一致，此处只给出参考的配置结果，记得同时修改 *iptables* 和 *resolv.conf*。


##### 4.4.1 主配置 *named.conf*
```
$ sudo vim /etc/named.conf 
options {
        directory       "/var/named";
        dump-file       "/var/named/data/cache_dump.db";
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt";
        allow-query     { any; };
        recursion yes;
};

zone "." IN {
        type hint;
        file "named.ca";
};

zone "wiki.centos.leannmak" IN {
        type master;
        file "named.wiki.centos.leannmak";
};

zone "182.168.192.in-addr.arpa" IN {
        type master;
        file "named.192.168.182";
};
```


##### 4.4.2 正解 *named.wiki.centos.leannmak*

```
$ sudo vim /var/named/named.centos.leannmak
$TTL    600
@       IN      SOA     dns.wiki.centos.leannmak. root.wiki.centos.leannmak.(
        2015073101 3H 15M 1W 1D)
@       IN      NS      dns.wiki.centos.leannmak.
dns.wiki.centos.leannmak.       IN      A       192.168.182.15
www.wiki.centos.leannmak.       IN      A       192.168.182.15
@       IN      MX      10      www.wiki.centos.leannmak.

www.wiki.centos.leannmak.    IN      A       192.168.182.15
linux.wiki.centos.leannmak.  IN      CNAME   www.wiki.centos.leannmak.
ftp.wiki.centos.leannmak.    IN      CNAME   www.wiki.centos.leannmak.
forum.wiki.centos.leannmak.  IN      CNAME   www.wiki.centos.leannmak.
```


##### 4.4.3 反解 *named.192.168.182*

```
$ sudo vim /var/named/named.192.168.182
$TTL    600
@       IN      SOA     dns.wiki.centos.leannmak. root.wiki.centos.leannmak.   (
                        2015073101      3H      15M     1W      1D      )
@       IN      NS      dns.wiki.centos.leannmak.
@       IN      NS      slave.wiki.centos.leannmak.
15      IN      PTR     dns.wiki.centos.leannmak.
15      IN      PTR     www.wiki.centos.leannmak.
```


##### 4.4.4 启动或重启DNS

配置完成后，记得启动或重新启动named服务，并测试及查看相关信息：

```
$ sudo service named start
$ sudo chkconfig named on

$ sudo tail -n 30 /var/log/messages | grep named    
...
Aug  2 22:28:35 cloudlab015 named[32326]: zone wiki.centos.leannmak/IN: loaded serial 2015073101
...

# 检查响应是否正确
$ sudo dig www.wiki.centos.leannmak @192.168.182.64
...
;; ANSWER SECTION:
www.wiki.centos.leannmak. 600   IN      A       192.168.182.15
...
```

参考文献：http://vbird.dic.ksu.edu.tw/linux_server/0350dns.php