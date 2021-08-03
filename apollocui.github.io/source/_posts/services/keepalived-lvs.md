---
layout: post
title: keepalived-lvs
date: 2019-07-08 13:17:45
tags:
    - [ keepalived, lvs ]
categories:
    - [ services ]
---
keepalived+lvs+nginx(DR)
<!-- more -->
## keepalived+lvs 工作原理
```
说起lvs，不得不说说关于lvs的工作原理，通常来说lvs的工作方式有三种：nat模式（LVS/NAT),直接路由模式（ LVS/DR），ip隧道模式(LVS/TUN),不过据说还有第四种模式（FULL NAT），下面我们来介绍介绍关于lvs常用的三种工作模式说明
```
### LVS负载均衡模式---NAT模式原理
````
LVS-NAT模式:NAT用法本来是因为网络IP地址不足而把内部保留IP地址通过映射转换成公网地址的一种上网方式(原地址NAT)如果把NAT的过程稍微变化,就可以 成为负载均衡的一种方式原理其实就是把从客户端发来的IP包的IP头目的地址在DR上换成其中一台REALSERVER的IP地址并发至此 REALSERVER,而REALSERVER则在处理完成后把数据经过DR主机发回给客户端,DR在这个时候再把数据包的原IP地址改为DR接口上的 IP地址即可期间,无论是进来的流量,还是出去的流量,都必须经过DR
````
如下为lvs nat模式的示意说明图：
<img src="/images/service/keepalived/lvs_nat.png" width=100% height=50% align=left/>
### DR(路由)
````
LVS-DR模式：每个Real Server上都有两个IP：VIP和RIP，但是VIP是隐藏的，就是不能提高解析等功能，只是用来做请求回复的源IP的，Director上只需要一个网卡，然后利用别名来配置两个IP：VIP和DIP，在DIR接收到客户端的请求后，DIR根据负载算法选择一台rs sever的网卡mac作为客户端请求包中的目标mac，通过arp转交给后端rs serve处理，后端再通过自己的路由网关回复给客户端
````
如下为lvs tun模式的示意说明图：
<img src="/images/service/keepalived/lvs_tun.png" width=100% height=50% align=left/>
### TUN(隧道)
````
LVS-TUN模式：它的连接调度和管理与VS/NAT中的一样，利用ip隧道技术的原理，即在原有的客户端请求包头中再加一层IP Tunnel的包头ip首部信息，不改变原来整个请求包信息，只是新增了一层ip首部信息，再利用路由原理将请求发给RS server，不过要求的是所有的server必须支持"IPTunneling"或者"IP Encapsulation"协议
````
## keepalived + lvs 
### 主和从上安装keepalived
````
yum -y install keepalived ipvsadm 
````
### keepalived.conf 配置master
````
! Configuration File for keepalived
global_defs {
   router_id lvs01          #router_id 机器标识，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
}
vrrp_instance VI_1 {            #vrrp实例定义部分
    state MASTER               #设置lvs的状态，MASTER和BACKUP两种，必须大写 
    interface ens160               #设置对外服务的接口
    virtual_router_id 100        #设置虚拟路由标示，这个标示是一个数字，同一个vrrp实例使用唯一标示 
    priority 100               #定义优先级，数字越大优先级越高，在一个vrrp——instance下，master的优先级必须大于backup
    advert_int 1              #设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    nopreempt
    authentication {           #设置验证类型和密码
        auth_type PASS         #主要有PASS和AH两种
        auth_pass 1111         #验证密码，同一个vrrp_instance下MASTER和BACKUP密码必须相同
    }
    virtual_ipaddress {         #设置虚拟ip地址，可以设置多个，每行一个
        192.168.3.199 
    }
}
virtual_server 192.168.3.199 88 {       #设置虚拟服务器，需要指定虚拟ip和服务端口
    delay_loop 6                 #健康检查时间间隔
    lb_algo wrr                  #负载均衡调度算法
    lb_kind DR                   #负载均衡转发规则
    persistence_timeout 50        #设置会话保持时间，对动态网页非常有用
    protocol TCP               #指定转发协议类型，有TCP和UDP两种
    real_server 192.168.3.105 88 {    #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 1               #设置权重，数字越大权重越高
    TCP_CHECK {              #realserver的状态监测设置部分单位秒
       connect_timeout 10       #连接超时为10秒
       retry 3             #重连次数
       delay_before_retry 3        #重试间隔
       connect_port 88         #连接端口为80，要和上面的保持一致
       }
    }
     real_server 192.168.3.114 88 {    #配置服务器节点1，需要指定real server的真实IP地址和端口
     weight 1                  #设置权重，数字越大权重越高
     TCP_CHECK {               #realserver的状态监测设置部分单位秒
       connect_timeout 10         #连接超时为10秒
       retry 3               #重连次数
       delay_before_retry 3        #重试间隔
       connect_port 88          #连接端口为80，要和上面的保持一致
       }
     }
}

````

### keepalived 配置backup
````
! Configuration File for keepalived
global_defs {
   router_id lvs02          #router_id 机器标识，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
}
vrrp_instance VI_1 {            #vrrp实例定义部分
    state BACKUP              #设置lvs的状态，MASTER和BACKUP两种，必须大写
    interface ens160           #设置对外服务的接口
    virtual_router_id 100         #设置虚拟路由标示，这个标示是一个数字，同一个vrrp实例使用唯一标示
    priority 99             #定义优先级，数字越大优先级越高，在一个vrrp——instance下，master的优先级必须大于backup
    advert_int 1              #设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    authentication {            #设置验证类型和密码
        auth_type PASS         #主要有PASS和AH两种
        auth_pass 1111         #验证密码，同一个vrrp_instance下MASTER和BACKUP密码必须相同
    }
    virtual_ipaddress {         #设置虚拟ip地址，可以设置多个，每行一个
        192.168.3.199
    }
}
virtual_server 192.168.3.199 88 {      #设置虚拟服务器，需要指定虚拟ip和服务端口
    delay_loop 6             #健康检查时间间隔
    lb_algo wrr              #负载均衡调度算法
    lb_kind DR               #负载均衡转发规则
    #persistence_timeout 50          #设置会话保持时间，对动态网页非常有用
    protocol TCP              #指定转发协议类型，有TCP和UDP两种
    real_server 192.168.3.105 88 {       #配置服务器节点1，需要指定real server的真实IP地址和端口
    weight 1                #设置权重，数字越大权重越高
    TCP_CHECK {              #realserver的状态监测设置部分单位秒
       connect_timeout 10         #连接超时为10秒
       retry 3                #重连次数
       delay_before_retry 3       #重试间隔
       connect_port 88           #连接端口为80，要和上面的保持一致
       }
    }
     real_server 192.168.3.114 88 {   #配置服务器节点1，需要指定real server的真实IP地址和端口
     weight 1                #设置权重，数字越大权重越高
     TCP_CHECK {              #realserver的状态监测设置部分单位秒
       connect_timeout 10          #连接超时为10秒
       retry 3             #重连次数
       delay_before_retry 3        #重试间隔
       connect_port 88         #连接端口为80，要和上面的保持一致
       }
     }
}

service keepalived start 
观察系统日志
````

### 两台 real server 部署一下脚本(放到/etc/init.d/下 )
````
[root@192-168-3-105 init.d]# cat realserver 
#!/bin/bash
# Script to start LVS DR real server.
# description: LVS DR real server
. /etc/rc.d/init.d/functions
VIP=192.168.3.199 #修改为VIP
host=`/usr/bin/hostname`
Ifconfig=`which ifconfig`
case "$1" in
    start)
        # Start LVS-DR real server on this machine.
        $Ifconfig lo down
        $Ifconfig lo up
        echo 1 > /proc/sys/net/ipv4/conf/lo/arp_ignore
        echo 2 > /proc/sys/net/ipv4/conf/lo/arp_announce
        echo 1 > /proc/sys/net/ipv4/conf/all/arp_ignore
        echo 2 > /proc/sys/net/ipv4/conf/all/arp_announce
        /sbin/ifconfig lo:0 $VIP broadcast $VIP netmask 255.255.255.255 up
        /sbin/route add -host $VIP dev lo:0
        ;; 
    stop)
        # Stop LVS-DR real server loopback device(s).
        $Ifconfig lo:0 down
        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_ignore
        echo 0 > /proc/sys/net/ipv4/conf/lo/arp_announce
        echo 0 > /proc/sys/net/ipv4/conf/all/arp_ignore
        echo 0 > /proc/sys/net/ipv4/conf/all/arp_announce
        ;; 
    status)

        # Status of LVS-DR real server.
        islothere=`$Ifconfig lo:0 | grep $VIP`
        isrothere=`netstat -rn | grep "lo:0" | grep $VIP`
        if [ ! "$islothere" -o ! "isrothere" ];then
            # Either the route or the lo:0 device
            # not found.
            echo "LVS-DR real server Stopped." 
        else
            echo "LVS-DR real server Running." 
        fi
        ;; 
    *) 
    # Invalid entry.
    echo "$0: Usage: $0 {start|status|stop}"
    exit 1
    ;; 
esac
exit 0
````
````
chmod 777 /etc/init.d/realserver

/etc/init.d/realserver start 
````

### 两台真是服务器安装nginx
````
yum -y install nginx 

把两台相应IP写到index.html 文件中(有标识行)
````

### 测试
````
关闭一台keepalived 

浏览器访问
http://192.168.3.199:88
````
## keepalived + lvs + nginx

### 配置描述 
```
keepalived和nginx部署在一台机器上
环境:centos7 
```
### 安装keepalived 和 nginx
````
yum -y install nginx keepalived ipvsadm
````
### keepalived master
````
! Configuration File for keepalived
global_defs {
   router_id lvs01          #router_id 机器标识，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
}
vrrp_instance VI_1 {            #vrrp实例定义部分
    state MASTER               #设置lvs的状态，MASTER和BACKUP两种，必须大写
    interface ens160               #设置对外服务的接口
    virtual_router_id 100        #设置虚拟路由标示，这个标示是一个数字，同一个vrrp实例使用唯一标示
    priority 100               #定义优先级，数字越大优先级越高，在一个vrrp——instance下，master的优先级必须大于backup
    advert_int 1              #设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    nopreempt
    authentication {           #设置验证类型和密码
        auth_type PASS         #主要有PASS和AH两种
        auth_pass 1111         #验证密码，同一个vrrp_instance下MASTER和BACKUP密码必须相同
    }
    virtual_ipaddress {         #设置虚拟ip地址，可以设置多个，每行一个
        192.168.3.199
    }
}
````

### keepalived backup
````
! Configuration File for keepalived
global_defs {
   router_id lvs02          #router_id 机器标识，通常为hostname，但不一定非得是hostname。故障发生时，邮件通知会用到。
}
vrrp_instance VI_1 {            #vrrp实例定义部分
    state BACKUP              #设置lvs的状态，MASTER和BACKUP两种，必须大写
    interface ens160           #设置对外服务的接口
    virtual_router_id 100         #设置虚拟路由标示，这个标示是一个数字，同一个vrrp实例使用唯一标示
    priority 99             #定义优先级，数字越大优先级越高，在一个vrrp——instance下，master的优先级必须大于backup
    advert_int 1              #设定master与backup负载均衡器之间同步检查的时间间隔，单位是秒
    authentication {            #设置验证类型和密码
        auth_type PASS         #主要有PASS和AH两种
        auth_pass 1111         #验证密码，同一个vrrp_instance下MASTER和BACKUP密码必须相同
    }
    virtual_ipaddress {         #设置虚拟ip地址，可以设置多个，每行一个
        192.168.3.199
    }
}
````

### nginx 配置(两台一样配置)
````
upstream uni_gateway_8082 {
        ip_hash;
        server 192.168.3.114:8082 weight=5 max_fails=3 fail_timeout=10;
        server 192.168.3.105:8082 weight=5 max_fails=3 fail_timeout=10;
}
    server {
        listen       8082;
        access_log  /var/log/nginx/access-89.log  main;
        location / {
                proxy_pass http://uni_gateway_8082;
                proxy_redirect  off;
                proxy_http_version 1.1;
                proxy_set_header  Host  $host;
                proxy_set_header  X-Real-IP $remote_addr;
                proxy_set_header  X-Forwarded-For $proxy_add_x_forwarded_for;
        }

    }
````
### vip 测试
````
通过关闭nginx 测试，漂移VIP
````
## 配置keepalived 日志
```
修改rsyslog 配置:
cat /etc/rsyslog.conf
*.info;mail.none;authpriv.none;cron.none;local0.none    /var/log/messages #新增local0.none
local0.*        /var/log/keepalived.log  #最下方新增这一行

重启rsyslog 和 keepalived
systemctl restart rsyslog 
systemctl restart keepalived 
```