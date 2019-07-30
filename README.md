lvs --(vip)haproxy(upstrem)--web(fastcgi)--php(解析代码)--mysql/redis

# 文件结构
.
├── ansible.cfg
├── hosts
├── playbook
│   ├── lnmp_1.yaml （实现一键搭建lnmp）
│   ├── lnmp_2.yaml （实现一键搭建架构）
│   └── mysql_db.yaml （实现创建数据库和用户）
└── roles
    ├── lvs
    │   ├── files
    │   │   └── keepalived.conf.j2
    │   ├── handlers
    │   │   └── main.yaml
    │   └── tasks
    │       ├── deploy_lvs.yaml
    │       └── install_lvs.yaml
    ├── mysql
    │   ├── files
    │   │   └── my.cnf.j2
    │   ├── handlers
    │   └── tasks
    │       ├── create_mysqldb.yaml
    │       └── install_mysql.yaml
    ├── nginx
    │   ├── files
    │   │   ├── nginx.conf
    │   │   └── realserver.sh
    │   ├── handlers
    │   └── tasks
    │       └── install_nginx.yaml
    └── web
        ├── files
        │   └── nginx.conf
        ├── handlers
        │   └── main.yaml
        └── tasks
            ├── copy_conf.yaml
            ├── install_nginx.yaml
            ├── install_php-fpm.yaml
            └── install_wordpress.yaml

# 部署lvs1与lvs2
1.安装软件和依赖
yum install ipvsadm keepalived gcc openssl openssl-devel

2.配置虚拟网卡设备（vip）
ifconfig ens33:0 172.16.13.8 broadcast 172.16.13.8 netmask 255.255.255.0 up

3.开启路由转发功能
echo 1 > /proc/sys/net/ipv4/ip_forward
其实在DR模式中，开启系统的包转发功能不是必须的，而在NAT模式下此操作是必须的。

4.开始配置ipvs
ipvsadm -C
ipvsadm -A -t 172.16.13.8:80 -s rr -p 600
ipvsadm -a -t 172.16.13.8:80 -r 172.16.13.6:80 -g
ipvsadm -a -t 172.16.13.8:80 -r 172.16.13.7:80 -g

第一行是清除内核虚拟服务器列表中的所有记录
第二行是添加一条新的虚拟IP记录，虚拟IP是172.16.13.8，同时指定持续服务时间为600秒。三四行是在新加虚拟IP记录中添加两条新的RS记录，并且指定LVS 的工作模式为直接路由模式。

5.配置keepalived.conf
其中在keepalive配置文件中，nopreempt  设置为不抢夺VIP

6.启动lvs和keepalive服务，开启自启动

# 部署nginx1和nginx2（实现七层负载
1.安装镜像源和软件
http://nginx.org/packages/centos/7/noarch/RPMS/nginx-release-centos-7-0.el7.ngx.noarch.rpm 
yum install -y nginx

2.配置upsteam模块到nginx，添加proxy_pass
upstream backend {
    server 172.16.13.10    weight=5;
}

server {
    location / {
        proxy_pass http://172.16.13.10;
    }
}

3.配置lo虚拟端口（这里写了个shell文件实现
```
#!/bin/bash
#description: Config realserver

VIP=172.16.13.8

case "$1" in
start)
       /sbin/ifconfig lo:0 $VIP netmask 255.255.255.255 broadcast $VIP
       /sbin/route add -host $VIP dev lo:0
       echo "1" >/proc/sys/net/ipv4/conf/lo/arp_ignore
       echo "2" >/proc/sys/net/ipv4/conf/lo/arp_announce
       echo "1" >/proc/sys/net/ipv4/conf/all/arp_ignore
       echo "2" >/proc/sys/net/ipv4/conf/all/arp_announce
       sysctl -p >/dev/null 2>&1
       echo "RealServer Start OK"
       ;;
stop)
       /sbin/ifconfig lo:0 down
       /sbin/route del $VIP >/dev/null 2>&1
       echo "0" >/proc/sys/net/ipv4/conf/lo/arp_ignore
       echo "0" >/proc/sys/net/ipv4/conf/lo/arp_announce
       echo "0" >/proc/sys/net/ipv4/conf/all/arp_ignore
       echo "0" >/proc/sys/net/ipv4/conf/all/arp_announce
       echo "RealServer Stoped"
       ;;
*)
       echo "Usage: $0 {start|stop}"
       exit 1
esac

exit 0

```

# 部署web服务器(nginx+php+web)
1.安装nginx
yum -y install nginx
2.配置nginx.conf
server{
    listen 80 default;
    server_name yanhua.com;
    index index.html index.htm index.php;
    root /data/www;
    location ~ .*\.(php|php5)?$
    {
      fastcgi_pass  127.0.0.1:9000;
      fastcgi_index index.php;
      include fastcgi_params;
      fastcgi_param SCRIPT_FILENAME /data/www$fastcgi_script_name;
    }
  }
  
3.创建根目录文件以及加权限
mkdir -p /data/www/
chmod 755 /data/www/
chown -R nginx.nginx /data/www/

4.安装php
更新yum源  http://rpms.remirepo.net/enterprise/remi-release-7.rpm 
下载软件 php php72-php-fpm php72-php-gd php72-php-json php72-php-mbstring php72-php-mysqlnd php72-php-xml php72-php-xmlrpc php72-php-opcache

5.启动服务 
systemctl start nginx
systemctl start php72-php-fpm

6.安装wordpress
下载 wget http://wordpress.org/latest.tar.gz
解压 tar -xzvf  latest.tar.gz

7.将wordpress内容拷贝到nginx的root路径上
cp -r /root/wordpress/*  /data/www/

8.重启nginx： systemctl restart nginx

9.配置连接数据库
mv /root/wordpress/wp-config-sample.php /root/wordpress/wp-config.php 
（将config文件加上数据库配置，或者在网页版上再进行数据库配置）

# 配置mysql
1.配置镜像源 
https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

2.将mysql5.7镜像加入源，安装
yum install -y mysql-community-server mysql-community-devel

3.添加wordpress数据库和数据库用户
创建数据库： CREATE DATABASE wordpress;
创建用户：
create user 'wordpress'@'localhost' identified by 'your password'; 
配置权限：
GRANT ALL PRIVILEGES ON wordpress.* TO wordpress@localhost IDENTIFIED BY 'your password';
刷新权限配置： FLUSH PRIVILEGES;
退出MySQL： QUIT;

# 踩的坑
1.安装上不懂镜像源，网上copy，要合适的软件找合适的镜像源
总结：#阿里巴巴镜像源 #163镜像源 #清华大学镜像源 #epel镜像源 #remi镜像源

2.配置lvs时候，没有出现虚拟vip
总结：
使用的是dr模式。vip地址是要取不存在的ip地址，如果是配置两台lvs+keepalived结构，虚拟vip会在两台机器上飘，需要调度器和下一层结构配置相应虚拟ip和虚拟lo接口

3.防火墙和selinux没有配置好
总结：
应该进行初始化机器，通过虚拟机的完整克隆。
在centos7上，
```
systemctl stop firewalld 
cp /etc/selinux/config /etc/selinux/config.bak
sed 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config.bak >/etc/selinux/config
```

4.学习ansible的role形式，和文本架构（tasks，files，handles
总结：
在执行ansible任务的时候，可以在playbook上形如：
```
include_role:
      name: lvs
      tasks_from: deploy_lvs
```

5.启动服务时候，centos6和centos7的模块不同
总结：
systemd模块 centos7
service模块 centos6

6.搭建lnmp的时候，html没有问题，php解析出现，file is not found
总结：原因有二
第一：主要参数fastcgi_param SCRIPT_FILENAME /data/www$fastcgi_script_name;
第二：Firewalls和selinux已关闭

7.ansible通过ssh连接某台机器，未连接上
总结：排查步骤
第一：ss -luntp 命令可以查看sshd服务占用端口
第二：端口已经改变，在ansible的hosts文件中加入参数 
如：172.16.13.6 ansible_ssh_port=52113
第三：ssh-copy-id -i ~/.ssh/id_rsa.pub  -p 端口号 root@172.16.13.6
第四：在/etc/ssh/sshd_config文件上，允许root登录 PermitRootLogin yes
第五：重启sshd服务

8.重启lvs失败
总结：排查步骤
第一：查看服务状态status
显示 Failed to start Initialise the Linux Virtual Server.
第二：分析原因，/etc/sysconfig/ipvsadm: 没有那个文件或目录，添加的lvs路线，没有保存到改文件下，导致重启不了。
第三：手动添加ipvsadm --save > /etc/sysconfig/ipvsadm

9.lnmp搭建时显示403
总结：
第一：403是权限拒绝的意思，应查看网页主目录权限
第二：权限分为`chown -R nginx.nginx`和 `chmod 755 /dir`

10.lnmp搭建显示502
总结：
第一：502表示服务器上的一个错误网关 ，因此说它是无效的，我们在出现了服务器502错误问题的时候，最好是先清除下缓存或者是在服务器上进行刷新试试的。
第二：重启服务，例如php-fpm等

11.出现网络不通，可以ping通网关，但dns解析有问题的现象
总结：排查思路

第一：ping网关，可通
第二：查看ifcfg文件，ping文件中的dns，不通 
第三：查看路由表，有无默认路由到网关，或者多几条路由。

存在问题，出错机器。
```
ip route
169.254.0.0/16 dev ens33 scope link metric 1002 
172.16.0.0/16 dev ens33 proto kernel scope link src 172.16.13.10 
```
解决：
```
删除多余路由，以及添加另一条默认路由
ip route add default via 172.16.13.2 dev ens33 
ip route del 169.254.0.0/16
```

12.mysql安装过后，进行初始化配置，无法进行远程登录
总结：
原因：远程登录时候需要root的原始密码，不能免密登录

解决方法：
第一：抓取mysql的root初始密码信息
```
grep 'temporary password' /var/log/mysqld.log 
|tail -n 1 | awk 'END {print $11}'
```
第二：在ansible上，注册register参数
第三：将下方写入my.cnf.j2文件
```
[client]
host=localhost
user=root
password={{outer_item}} 
```
第四：template到远程。实现mysql直接登录。

