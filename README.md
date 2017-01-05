#脚本说明
##编译安装

这里展示脚本文件的一部分

```
tar -xf $INSTALL_FILE
cd $CODE_DIR
[ -f configure ] && ./configure --prefix=/usr/local/varnish  &> $NULL || exit 40
echo "configure success."
make &> $NULL
[ $? -eq 0 ] && echo "make success."  || exit 41
make install &> $NULL
[ $? -eq 0 ] && echo "make install success." || exit 42
```

由于这是源码包安装，所以安装后并不能像其他服务那样会在etc下有相应的配置文件等，
虽然这些文件在源码包中都有，所以很明显我们需要将一些对应的文件进行操作。

```
##进行配置文件的复制操作；
cp redhat/varnish.initrc /etc/init.d/varnish
cp redhat/varnish.sysconfig  /etc/sysconfig/varnish
cp redhat/varnish_reload_vcl /usr/bin/
ln -s /usr/local/varnish/sbin/varnishd /usr/sbin/
mkdir /etc/varnish
cp /usr/local/varnish/etc/varnish/default.vcl /etc/varnish/
uuidgen > /etc/varnish/secret
```

###文件的具体说明：

- /etc/varnish/  ：配置文件目录
  - /etc/init.d/varnish :varnish的启动程序
  - /etc/sysconfig/varnish :配置文件，varnish定义自身属性
  - /etc/varnish/default.vcl ：默认配置文件，定义后端节点的
- /usr/bin/varnish_reload_vcl ：加载vcl
- /usr/bin/varnishadm ： 客户端程序
- /usr/bin/varnishstat ：状态监控

###配置文件参数说明：

```
[root@svr5 ~]# vim /etc/sysconfig/varnish
VARNISH_VCL_CONF=/etc/varnish/default.vcl                #vcl文件路径
VARNISH_LISTEN_PORT=80                                #默认端口
VARNISH_SECRET_FILE=/etc/varnish/secret                #密钥文件
VARNISH_STORAGE_SIZE=64M                                #缓存大小
VARNISH_STORAGE="malloc,${VARNISH_STORAGE_SIZE}"        #基于内存方式

[root@svr5 ~]# vim  /etc/varnish/default.vcl
backend default {
     .host = "192.168.4.205";	##后端web服务器的ip
     .port = "80";		##后端web服务器的httpd使用的端口号
 }
```

##操作实例

###实例需求

通过配置Varnish缓存服务器，实现如下目标：

- 使用Varnish加速后端Apache Web服务
- 使用varnishadm管理缓存页面
- 使用varnishstat查看Varnish状态


###具体操作

使用3台RHEL6虚拟机，其中一台Web服务器，一台Varnish代理服务器，一台作为测试用的Linux客户机。
这几台服务器都不需要配网关，客户端计算机和web服务器不能之间ping通，具体的ip地址见下文。

首先在进行web服务器的搭建，这里我们进行最简单的httpd的服务器就可以了，以便于测试。

```
[root@web02 ~]# ifconfig eth1
eth1      Link encap:Ethernet  HWaddr 54:52:01:01:16:02  
          inet addr:192.168.2.16  Bcast:192.168.2.255  Mask:255.255.255.0
[root@web02 ~]# yum clean all
[root@web02 ~]# yum repolist
...
repolist: 3,819
[root@web02 ~]# yum -y install httpd
[root@web02 ~]# echo "This is index.html." > /var/www/html/index.html
[root@web02 ~]# service httpd restart
停止 httpd：                                               [失败]
正在启动 httpd：httpd: apr_sockaddr_info_get() failed for web02.wolf.cn
httpd: Could not reliably determine the server's fully qualified domain name, using 127.0.0.1 for ServerName
                                                           [确定]
[root@web02 ~]# chkconfig httpd on
//测试web服务，测试web服务正常
[root@web02 ~]# curl http://192.168.2.16
This is index.html.

```

然后进行代理服务器的配置和搭建，这里使用自己编写的脚本进行服务的安装和部署，
脚本文件和源码包都放在gethub上了。

```
[root@proxy02 varnish]# ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 54:52:01:01:13:01  
          inet addr:192.168.4.6  Bcast:192.168.4.255  Mask:255.255.255.0
[root@proxy02 varnish]# ifconfig eth1
eth1      Link encap:Ethernet  HWaddr 54:52:01:01:13:02  
          inet addr:192.168.2.6  Bcast:192.168.2.255  Mask:255.255.255.0
[root@proxy02 varnish]# ll
total 2008
-rwxr-xr-x. 1 root root    2969 Jan  5 12:27 install_varnish.sh
-rw-r--r--. 1 root root 2049810 Jan  5 09:47 varnish-3.0.6.tar.gz
[root@proxy02 varnish]# ./install_varnish.sh 
configure success.
make success.
make install success.
//对于对应的配置文件的修改，都在脚本中使用sed进行修改，
//在这里就直接开启服务，并设置为开机启动即可。
[root@proxy02 varnish]# service varnish restart
Stopping Varnish Cache:                                    [确定]
Starting Varnish Cache:                                    [确定]
[root@proxy02 varnish]# chkconfig varnish on
[root@proxy02 varnish]# chkconfig varnish --list
varnish        	0:off	1:off	2:on	3:on	4:on	5:on	6:off
```

最后进行客户端的测试即可。

```
[root@client02 ~]# ifconfig eth0
eth0      Link encap:Ethernet  HWaddr 54:52:01:01:15:01  
          inet addr:192.168.4.15  Bcast:192.168.4.255  Mask:255.255.255.0
//这里就可以看到我们访问的是代理服务器的ip就可以访问web服务
[root@client02 ~]# curl http://192.168.4.6
This is index.html.
```

当对网页的信息更新频率要求很高时，就可以使用下列命令进行设置
[root@proxy02 varnish]#ln -s /usr/local/varnish/bin/* /usr/bin/
[root@proxy02 varnish]#varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082 ban.url index.html
[root@proxy02 varnish]#varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082 ban.url ".*"
