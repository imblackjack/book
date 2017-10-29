```
[root@server bin]# cd /usr/local/src/
```

```
[root@server src]# ls
[root@server src]# git clone https://source.isc.org/git/bind9.git
Cloning into 'bind9'...
remote: Counting objects: 454545, done.
remote: Compressing objects: 100% (76149/76149), done.
Receiving objects: 100% (454545/454545), 215.97 MiB | 8.31 MiB/s, done.
remote: Total 454545 (delta 375727), reused 453100 (delta 374590)
Resolving deltas: 100% (375727/375727), done.
[root@server src]# ls
bind9
```

创建named用户

```
[root@server bin]# group -r -g 53 named
[root@server bin]# useradd -r -u 53 -g 53 named
```

编译安装

```
[root@server src]# cd bind9/
[root@server bind9]# ./configure --prefix=/usr/local/bind9 --sysconfdir=/etc/named/ --disable-chroot --enable-threads --without-openssl
#指明安装位置，配置文件位置，关闭chroot，开启线程,不使用openssl
[root@server bind9]# make && make install
```

自行编译bind源码包会产生以下问题:  
（1）没有配置文件  
（2）没有区域解析文件（包括13个根服务器的解析文件）  
（3）没有rndc的相关配置文件

解决上述问题

```
#1、将bind下配置文件加入PATH中
[root@server bind9]# vim /etc/profile.d/named.sh
export PATH=/usr/local/bind9/bin:/usr/local/bind9/sbin:$PATH
[root@server bind9]# . /etc/profile.d/named.sh

#2、导出库文件搜索路径
[root@server bind9]# vim /etc/ld.so.conf.d/named.conf
/usr/local/bind9/lib
[root@server bind9]# ldconfig -v

#3、导出头文件搜索路径
[root@server bind9]# ln -sv /usr/local/bind9/include /usr/include/named
"/usr/include/named" -> "/usr/local/bind9/include"

#4、导出帮助文档搜索路径
[root@server bind9]# vim /etc/man_db.conf
添加一行
MANDATORY_MANPATH                       /usr/local/bind9/share/man
```

编辑主配置文件  
\[root@server bind9\]\# vim /etc/named/named.conf

```
//named.conf

options {
    listen-on port 53 { 127.0.0.1; 172.17.119.82; };
//    listen-on-v6 port 53 { ::1; };
    directory   "/var/named";
    allow-query     { any; };
    recursion yes;
};
//根解析区域
zone "." IN {
        type hint;
        file "named.ca";//需自行创建
};
//正向解析区域
zone "localhost" IN {
        type master;
        file "locaihost.zone";//需自行创建
        allow-update { none; };
};
//反向解析区域
zone "0.0.127.in-addr.arpa" IN {
        type master;
        file "named.local";//需自行创建
        allow-update { none; };
};

//isurecloud.com正向解析区域
zone "isurecloud.com" IN {
        type master;
        file "isurecloud.com.zone";//需自行创建
        allow-update { none; };
};
```

创建区域解析库文件

```
[root@server named]# vim localhost.zone
$TTL 86400
@       IN      SOA     localhost.      admin.localhost. (
                        2016091301
                        1H
                        5M
                        7D
                        1D )
        IN      NS      localhost.
localhost.      IN      A       127.0.0.1

[root@server named]# vim named.local
$TTL 86400
@       IN      SOA     localhost.      admin.localhost. (
                        2016091301
                        1H
                        5M
                        7D
                        1D )
        IN      NS      localhost.
1       IN      PTR     localhost.

#在联网的情况下直接将查询根的结果导入根区域配置文件
[root@server named]# dig -t NS . > /var/named/named.ca

#检查zone文件
[root@server named]# named-checkzone "isurecloud.com" /var/named/isurecloud.com.zone
zone isurecloud.com/IN: loaded serial 2017061301
OK
[root@server named]# named-checkzone "localhost" /var/named/localhost.zone
zone localhost/IN: loaded serial 2016091301
OK
```

接下来，为了安全，需要更改主配置文件和解析库文件的权限和属组

```
[root@server named]# chmod 640 isurecloud.com.zone  localhost.zone  named.ca named.local
[root@server named]# chown :named isurecloud.com.zone  localhost.zone  named.ca named.local
[root@server named]# chmod 640 /etc/named/named.conf
[root@server named]# chown :named /etc/named/named.conf
```

尝试启动bind

```
[root@server named]# named -u named
[root@server named]# echo $?
1
#发现启动失败，查看详细日志
[root@server named]# tail -f /var/log/messages

Oct 25 18:58:06 server named[1871]: ----------------------------------------------------
Oct 25 18:58:06 server named[1871]: BIND 9 is maintained by Internet Systems Consortium,
Oct 25 18:58:06 server named[1871]: Inc. (ISC), a non-profit 501(c)(3) public-benefit
Oct 25 18:58:06 server named[1871]: corporation.  Support and training for BIND 9 are
Oct 25 18:58:06 server named[1871]: available at https://www.isc.org/support
Oct 25 18:58:06 server named[1871]: ----------------------------------------------------
Oct 25 18:58:06 server named[1871]: adjusted limit on open files from 65535 to 1048576
Oct 25 18:58:06 server named[1871]: found 1 CPU, using 1 worker thread
Oct 25 18:58:06 server named[1871]: using 1 UDP listener per interface
Oct 25 18:58:06 server named[1871]: using up to 4096 sockets
Oct 25 18:58:06 server named[1871]: loading configuration from '/etc/named/named.conf'
Oct 25 18:58:06 server named[1871]: directory '/var/named' is not writable
Oct 25 18:58:06 server named[1871]: /etc/named/named.conf:6: parsing failed: permission denied
Oct 25 18:58:06 server named[1871]: loading configuration: permission denied
Oct 25 18:58:06 server named[1871]: exiting (due to fatal error)

#原来是因为named用户对/var/named/目录没有写权限，于是修改权限
[root@server named]# chmod 770  /var/named/
[root@server named]# named -u named
[root@server named]# echo $?
0

#查看是否启动成功
[root@server ~]# ss -unlp |grep :53
UNCONN     0      0      172.17.119.82:53                       *:*                   users:(("named",pid=2084,fd=514))
UNCONN     0      0      127.0.0.1:53                       *:*                   users:(("named",pid=2084,fd=513))
UNCONN     0      0           :::53                      :::*                   users:(("named",pid=2084,fd=512))
#启动成功，已监听53端口
```



