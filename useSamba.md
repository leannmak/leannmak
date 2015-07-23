## ���ʹ��Sambaʵ��Linux��Windows����

by leannmak  2015-7-24

### 1. ��Linux�ϰ�װSamba:

* ���Linux���Ƿ��Ѱ�װSamba��
```bash
$ sudo rpm -qa samba
samba-3.6.23-14.el6_6.x86_64
```
 �������Ͻ������Samba�Ѱ�װ��

* ����ѯ���Ϊ�գ���ʹ�� `yum` ��װ��
```
$ sudo yum install samba samba-client
```

* ʹ��`vim`�޸�Samba����:
```
$ sudo vim /etc/samba/smb.conf
```

1) �޸� `security=user` Ϊ `security=share`

2) ���ļ�ĩβ����������ݣ�
```
[share]
comment = leannmak's Files Share
path = /home/leannmak/share
browseable = yes
guest ok = yes
writable = yes
```
`path` ����ΪLinux��Windows�Ĺ���Ŀ¼��

* ��Linux�д��� `path` Ŀ¼����ȷ����Ŀ¼���ڣ�
```
$ cd /home/leannmak
$ mkdir share
```

* �޸Ĺ���Ŀ¼Ȩ�ޣ�ȷ��Windows�������ܲ�����Ŀ¼�µ��ļ���
```
$ sudo chmod -R 777 share
```
`-R` ��ʾ�Ը�Ŀ¼�µ�������Ŀ¼�ݹ�ִ�иò�����

* �޸���󣬲鿴Samba״̬��
```
$ sudo service smb status
smbd (pid  19087) is running...
$ sudo service smb restart
```
��Sambaδ������ִ������������ֱ��`start`��


### 2. ��Windows����������������:

* �� **��ʼ** �˵������� **����**������ `\\Samba������IP��ַ\share` ��

* �����ʲ��ɹ�����ִ�����²�����

1) ȷ��Windows�ķ���ǽ�ѹرգ�
������� --> Сͼ�� --> ������ --> ���� --> Windows Firewall�� �رա�

2) ���Samba������`selinux`��`iptables`:
* �޸�**selinux**�����ļ���
```
$ sudo vi /etc/sysconfig/selinux
```
�޸�SELINUX=enforcing(Ĭ��ֵ)ΪSELINUX=disabled���˳���ִ�У�
```
$ sudo setenforce 0
```
* �ر�Linux����ǽ��
```
$ sudo service iptables stop
```

* ��Windows ����� --> ӳ���������������� `\\samba������IP��ַ\share` ӳ��ΪZ:�̡�


### 3. ��Windows Z:�����½�һ��`leannmak.txt`�ļ�����Samba�����в鿴��
```
$ sudo cd /home/leannmak/share
$ sudo ll
```
��ʱ���Կ����½����ļ���Ϣ��Done��

### 4. ϵͳreboot�����ӳ����ܻ�����ô�죿

* Pls Make Sure that your **Firewalls** on both Windows & Linux have already **closed** !!! 
  Also **SELINUX** is **disabled** !!!
  

  
�ο�����: http://my.oschina.net/daisheng/blog/466520
