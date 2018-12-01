## Bind 搭建域名私服

### Linux安装

```
yum install -y bind bind-utils
```

```
[root@host ~]# yum install -y bind bind-utils
Loaded plugins: fastestmirror
Setting up Install Process
Determining fastest mirrors
 * base: mirror.scalabledns.com
 * extras: mirror.fileplanet.com
 * updates: repos-lax.psychz.net
base                                                     | 3.7 kB     00:00     
extras                                                   | 3.3 kB     00:00     
updates                                                  | 3.4 kB     00:00     
updates/primary_db                                       | 1.8 MB     00:00     
Resolving Dependencies
--> Running transaction check
---> Package bind.i686 32:9.8.2-0.68.rc1.el6_10.1 will be installed
--> Processing Dependency: bind-libs = 32:9.8.2-0.68.rc1.el6_10.1 for package: 32:bind-9.8.2-0.68.rc1.el6_10.1.i686
--> Processing Dependency: portreserve for package: 32:bind-9.8.2-0.68.rc1.el6_10.1.i686
--> Processing Dependency: liblwres.so.80 for package: 32:bind-9.8.2-0.68.rc1.el6_10.1.i686
--> Processing Dependency: libisccfg.so.82 for package: 32:bind-9.8.2-0.68.rc1.el6_10.1.i686
--> Processing Dependency: libisccc.so.80 for package: 32:bind-9.8.2-0.68.rc1.el6_10.1.i686
--> Processing Dependency: libisc.so.83 for package: 32:bind-9.8.2-0.68.rc1.el6_10.1.i686
--> Processing Dependency: libdns.so.81 for package: 32:bind-9.8.2-0.68.rc1.el6_10.1.i686
--> Processing Dependency: libbind9.so.80 for package: 32:bind-9.8.2-0.68.rc1.el6_10.1.i686
---> Package bind-utils.i686 32:9.8.2-0.68.rc1.el6_10.1 will be installed
--> Running transaction check
---> Package bind-libs.i686 32:9.8.2-0.68.rc1.el6_10.1 will be installed
---> Package portreserve.i686 0:0.0.4-11.el6 will be installed
--> Finished Dependency Resolution

Dependencies Resolved

================================================================================
 Package          Arch      Version                          Repository    Size
================================================================================
Installing:
 bind             i686      32:9.8.2-0.68.rc1.el6_10.1       updates      4.0 M
 bind-utils       i686      32:9.8.2-0.68.rc1.el6_10.1       updates      188 k
Installing for dependencies:
 bind-libs        i686      32:9.8.2-0.68.rc1.el6_10.1       updates      902 k
 portreserve      i686      0.0.4-11.el6                     base          23 k

Transaction Summary
================================================================================
Install       4 Package(s)

Total download size: 5.1 M
Installed size: 10 M
Downloading Packages:
(1/4): bind-9.8.2-0.68.rc1.el6_10.1.i686.rpm             | 4.0 MB     00:00     
(2/4): bind-libs-9.8.2-0.68.rc1.el6_10.1.i686.rpm        | 902 kB     00:00     
(3/4): bind-utils-9.8.2-0.68.rc1.el6_10.1.i686.rpm       | 188 kB     00:00     
(4/4): portreserve-0.0.4-11.el6.i686.rpm                 |  23 kB     00:00     
--------------------------------------------------------------------------------
Total                                            33 MB/s | 5.1 MB     00:00     
Running rpm_check_debug
Running Transaction Test
Transaction Test Succeeded
Running Transaction
  Installing : 32:bind-libs-9.8.2-0.68.rc1.el6_10.1.i686                    1/4 
  Installing : portreserve-0.0.4-11.el6.i686                                2/4 
  Installing : 32:bind-9.8.2-0.68.rc1.el6_10.1.i686                         3/4 
  Installing : 32:bind-utils-9.8.2-0.68.rc1.el6_10.1.i686                   4/4 
  Verifying  : portreserve-0.0.4-11.el6.i686                                1/4 
  Verifying  : 32:bind-utils-9.8.2-0.68.rc1.el6_10.1.i686                   2/4 
  Verifying  : 32:bind-9.8.2-0.68.rc1.el6_10.1.i686                         3/4 
  Verifying  : 32:bind-libs-9.8.2-0.68.rc1.el6_10.1.i686                    4/4 

Installed:
  bind.i686 32:9.8.2-0.68.rc1.el6_10.1                                          
  bind-utils.i686 32:9.8.2-0.68.rc1.el6_10.1                                    

Dependency Installed:
  bind-libs.i686 32:9.8.2-0.68.rc1.el6_10.1   portreserve.i686 0:0.0.4-11.el6  

Complete!
```

关键文件

```
[root@localhost ~]# rpm -ql bind
/etc/NetworkManager/dispatcher.d/13-named
/etc/logrotate.d/named
/etc/named    
/etc/named.conf    #bind主配置文件
/etc/named.iscdlv.key
/etc/named.rfc1912.zones    #定义zone的文件
/etc/named.root.key
/etc/portreserve/named
/etc/rc.d/init.d/named    #bind脚本文件
/etc/rndc.conf    #rndc配置文件
/etc/rndc.key
/etc/sysconfig/named
/usr/lib64/bind
/usr/sbin/arpaname
/usr/sbin/ddns-confgen
/usr/sbin/dnssec-dsfromkey
/usr/sbin/dnssec-keyfromlabel
/usr/sbin/dnssec-keygen
/usr/sbin/dnssec-revoke
/usr/sbin/dnssec-settime
/usr/sbin/dnssec-signzone
/usr/sbin/genrandom
/usr/sbin/isc-hmac-fixup
/usr/sbin/lwresd
/usr/sbin/named
/usr/sbin/named-checkconf    #检测/etc/named.conf文件语法
/usr/sbin/named-checkzone    #检测zone和对应zone文件的语法
/usr/sbin/named-compilezone
/usr/sbin/named-journalprint
/usr/sbin/nsec3hash
/usr/sbin/rndc    #远程dns管理工具
/usr/sbin/rndc-confgen    #生成rndc密钥

#过长省略

/var/log/named.log
/var/named
/var/named/data
/var/named/dynamic
/var/named/named.ca    #根解析库
/var/named/named.empty
/var/named/named.localhost    #本地主机解析库
/var/named/named.loopback    
/var/named/slaves    #从文件夹
/var/run/named
```

手动创建bind主配置文件

```
[root@localhost etc]# vim named.conf    #不熟悉的可以直接通过修改原始的配置文件
options {
  directory "/var/named";
 
  };    
zone "." IN {
  type hint;
  file "named.ca";
};
include "/etc/named.rfc1912.zones";
include "/etc/named.root.key";
```

定义一个zone

```
[root@localhost etc]# cat >> /etc/named.rfc1912.zones << EOF    #这里使用Here Document，不懂得可以搜素
> zone "anyisalin.com" IN {            #定义区域为anyisalin.com
>    type master;                    #设置类型为master
>    file "anyisalin.com.zone";        #解析库文件名称为anyisalin.com.zone
> };
> EOF
```

创建区域解析库文件

```
[root@localhost etc]# vim /var/named/anyisalin.com.zone 
$TTL 600    #定义全局默认超时时间
$ORIGIN anyisalin.com.    #定义后缀
@   IN  SOA    ns1.anyisalin.com.  admin.anyisalin.com. (
                20160321    #序列号
                1H    #刷新时间
                5M    #重试时间
                1W    #超时时间
                10M )    #否定答案缓存TTL值
        IN      NS      ns1
ns1     IN      A       192.168.192.150
        IN      MX 10   mail1
mail1   IN      A       192.168.192.1
www     IN      A       192.168.192.2
cname   IN      CNAME   www                #别名, 将cname.anyisalin.com. 解析到 www.anyisalin.com.的地址
*       IN      A       192.168.2.1    #泛域名解析，以上都不是的解析到192.168.2.1
```

检查、启动并测试

```
[root@localhost etc]# named-checkconf     #检查主配置文件语法
[root@localhost etc]# named-checkzone "anyisalin.com" /var/named/anyisalin.com.zone     #检查anyisalin.com zone所对应的解析库文件
zone anyisalin.com/IN: loaded serial 20160321
OK
[root@localhost etc]# service named start
Starting named:                                            [  OK  ]
```

http://blog.51cto.com/anyisalin/1753638