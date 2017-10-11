# DNS单台部署bind
矮哥linux运维群： 93324526 笔者QQ：57884322
转载： https://www.abcdocker.com/abcdocker/1298
8
---

##基础环境
```sh
[root@Aige ~]# cat /etc/redhat-release 
CentOS release 6.8 (Final)
[root@Aige ~]# uname -a
Linux Aige 2.6.32-642.el6.x86_64 #1 SMP Tue May 10 17:27:01 UTC 2016 x86_64 x86_64 x86_64 GNU/Linux
[root@Aige ~]# /etc/init.d/iptables stop
```
##安装
```
yum install bind-utils bind bind-devel bind-chroot -y
```
##编辑/etc/named.conf
```
[root@bogon ~]# >/etc/named.conf
[root@bogon ~]# vim /etc/named.conf   DNS首先加载的文件 更多ban9管理员
 options {
  version "1.1.1";
  listen-on port 53 {any;};
  directory "/var/named/chroot/etc/";
  pid-file "/var/named/chroot/var/run/named/named.pid";
  allow-query { any; };
  Dump-file "/var/named/chroot/var/log/binddump.db";
  Statistics-file "/var/named/chroot/var/log/named_stats";
  zone-statistics yes;
  memstatistics-file "log/mem_stats";
  empty-zones-enable no;
  forwarders {202.106.196.115;8.8.8.8; };
};
key "rndc-key" {
        algorithm hmac-md5;
        secret "Eqw4hClGExUWeDkKBX/pBg==";
};
controls {
       inet 127.0.0.1 port 953
               allow { 127.0.0.1; } keys { "rndc-key"; };
};
logging {
  channel warning {
    file "/var/named/chroot/var/log/dns_warning" versions 10 size 10m;
    severity warning;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  channel general_dns {
    file "/var/named/chroot/var/log/dns_log" versions 10 size 100m;
    severity info;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  category default {
    warning;
  };
  category queries {
    general_dns;
  };
};
include "/var/named/chroot/etc/view.conf";
```
###相关注释：
```
 options {
  version "1.1.1";
  listen-on port 53 {any;};     #监控端口
  directory "/var/named/chroot/etc/";   #A记录等配置文件所在的目录
  pid-file "/var/named/chroot/var/run/named/named.pid";  服务器来的进程号
  allow-query { any; };      #允许谁访问{}可以写IP
  Dump-file "/var/named/chroot/var/log/binddump.db";
  Statistics-file "/var/named/chroot/var/log/named_stats";  #可以看到DNS解析记录的数量，成功率有多少域（可以做监控）DNS状态等
  zone-statistics yes;        #配成yes之后上面才会写入
  memstatistics-file "log/mem_stats";     #内存状态
  empty-zones-enable no;         
  forwarders {202.106.196.115;8.8.8.8; };  转发（如果我这没有域名就转发）
};
key "rndc-key" {     认证的密钥
        algorithm hmac-md5;
        secret "Eqw4hClGExUWeDkKBX/pBg==";
};
controls {                      
       inet 127.0.0.1 port 953
               allow { 127.0.0.1; } keys { "rndc-key"; };
};
logging {                    日志  警告路径 （日志分2种）
  channel warning {           日志的相关信息
    file "/var/named/chroot/var/log/dns_warning" versions 10 size 10m;
    severity warning;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  channel general_dns {   访问日志相关信息
    file "/var/named/chroot/var/log/dns_log" versions 10 size 100m;
    severity info;
    print-category yes;
    print-severity yes;
    print-time yes;
  };
  category default {     #默认日志警告级别
    warning;
  };
  category queries {    #访问日志级别
    general_dns;
  };
};
include "/var/named/chroot/etc/view.conf";  其他域的配置文件目录(view.conf可以实现简单的只能DNS的功能，为以后的只能DNS做准备)
```
##编辑认证/etc/rndc.key
```
key "rndc-key" {
        algorithm hmac-md5;
        secret "Eqw4hClGExUWeDkKBX/pBg==";
};
```
##编辑/etc/rndc.conf
```
key "rndc-key" {
        algorithm hmac-md5;
        secret "Eqw4hClGExUWeDkKBX/pBg==";
};
options {
        default-key "rndc-key";
        default-server 127.0.0.1;
        default-port 953;
};
```
##编辑/var/named/chroot/etc/view.conf
```
view "View" {
  zone "lnh.com" {
        type    master;
        file    "lnh.com.zone";
        allow-transfer {
                10.255.253.211;
        };
        notify  yes;
        also-notify {
                10.255.253.211;
        };
  };
};
```
###相关注释
```
view "View" {    #“View 说明 以后可以修改为电信 联通”
  zone "lnh.com" {     #zone配置
        type    master;   #此处为master 还有一个slave
        file    "lnh.com.zone";  #通过这个view进行来找file 找哪一个
        allow-transfer {         #允许谁来找我要数据（在named.conf(allow-quey)配置）  允许谁来向我发送请求
                10.255.253.211;
        };
        notify  yes;     #当我的master文件发生变更了，去通知下面的IP，如
        also-notify {
                10.255.253.211;
        };
  };
};
如果有多个需要一zone"lnh.com开头"以};结尾
```
##编辑/var/named/chroot/etc/lnh.com.zone
```
$ORIGIN .
$TTL 3600       ; 1 hour
lnh.com                  IN SOA  op.lnh.com. dns.lnh.com. (
                                2000       ; serial
                                900        ; refresh (15 minutes)
                                600        ; retry (10 minutes)
                                86400      ; expire (1 day)
                                3600       ; minimum (1 hour)
                                )
                        NS      op.lnh.com.
$ORIGIN lnh.com.
shanks              A       1.2.3.4
op              A       1.2.3.4
```
###相关注释
```
$ORIGIN .     #zone文件生效的域
$TTL 3600       ; 1 hour   #域名生存周期（内网可以调节端，外网长）
lnh.com    对外说明我是那个zone           IN SOA  op.lnh.com. dns.lnh.com. (  对
                                2000       ; serial   #如果新同步数据需要调节此处的大小
                                900        ; refresh (15 minutes)   #当我slave向我同步数据，同步失败多久响应
                                600        ; retry (10 minutes)   #10分钟之后我在发出请求
                                86400      ; expire (1 day)      1年没有响应我就认为他宕机了
                                3600       ; minimum (1 hour)
                                )
                        NS      op.lnh.com.   此处A记录的相关信息
$ORIGIN lnh.com.               #上面NS配置的域名在下面一定要有一个A记录
shanks              A       1.2.3.4
op              A       1.2.3.4
```
###相关注释2
```
1、Serial：只是一个序号，但这个序号可被用来作为slave与master更新的依据。举例来说，master序号为100但slave序号为90时，那么这个zone file的资料就会被传送到slave来更新了。由于这个序号代表新旧资料，通常我们建立你可以利用日期来设定！ 
举例来说，上面的资料是在2016/01/01所修改的第一次，所以用2016010101作为序号代表！（yyymmddnn，nn代表这一天是第几次修改） 
2、Refresh：除了根据Serial来判断新旧之外，我们可以利用这个refresh（更新）命令slave多久进行一次主动更新； 
3、Retry：如果到了Refresh的时间，但是slave却无法连接到master时，那么在多久之后，slave会再次的主动尝试与主机连线； 
4、Expire：如果slave一直无法与master连接上，那么经过多久的时间之后，则命令slave不要再连接master了！也就是说，此时我们假设master DNS可能遇到重大问题而无法上线，则等待系统管理员处理完毕后，再重新来到slave DNS重新启动bind吧！ 
5、Minimun：这个就有点像是TTL
```
##修改默认权限并启动
```
 cd /var && chown -R named.named named/
/etc/init.d/named start              
chkconfig named on   
```
###相关注释
```
cd /var && chown -R named.named named/   授权目录
/etc/init.d/named start               启动服务
chkconfig named on             开机启动
```
##事后监查
检查服务启动情况（UDP 53 监控（2个））
```
[root@web02 var]# netstat -lntup
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address               Foreign Address             State       PID/Program name   
tcp        0      0 172.16.1.8:53               0.0.0.0:*                   LISTEN      35157/named         
tcp        0      0 10.0.0.8:53                 0.0.0.0:*                   LISTEN      35157/named         
tcp        0      0 127.0.0.1:53                0.0.0.0:*                   LISTEN      35157/named         
tcp        0      0 0.0.0.0:22                  0.0.0.0:*                   LISTEN      1329/sshd           
tcp        0      0 127.0.0.1:953               0.0.0.0:*                   LISTEN      35157/named         
tcp        0      0 :::22                       :::*                        LISTEN      1329/sshd           
udp        0      0 172.16.1.8:53               0.0.0.0:*                               35157/named         
udp        0      0 10.0.0.8:53                 0.0.0.0:*                               35157/named         
udp        0      0 127.0.0.1:53                0.0.0.0:*                               35157/named         
udp        0      0 0.0.0.0:44354               0.0.0.0:*                               35157/named    
```
##查看解析
dig @127.0.0.1 a.lnh.com
```
[root@web02 var]# dig @127.0.0.1 a.lnh.com
; <<>> DiG 9.8.2rc1-RedHat-9.8.2-0.47.rc1.el6 <<>> @127.0.0.1 a.lnh.com
; (1 server found)
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NXDOMAIN, id: 18014
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 0, AUTHORITY: 1, ADDITIONAL: 0
;; QUESTION SECTION:
;a.lnh.com.            IN    A
;; AUTHORITY SECTION:
lnh.com.        3600    IN    SOA    op.lnh.com. dns.lnh.com. 2000 900 600 86400 3600
;; Query time: 1 msec
;; SERVER: 127.0.0.1#53(127.0.0.1)
;; WHEN: Wed Aug 17 02:06:27 2016
;; MSG SIZE  rcvd: 70
```
如果不指定DNS要看本次的dns server服务器
```
[root@bogon var]# cat /etc/resolv.conf
nameserver 192.168.56.2
nameserver 8.8.8.8
search localhost
```