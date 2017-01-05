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

当对网页的信息更新频率要求很高时，就可以使用下列命令进行设置
[root@proxy02 varnish]#ln -s /usr/local/varnish/bin/* /usr/bin/
[root@proxy02 varnish]#varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082 ban.url index.html
[root@proxy02 varnish]#varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082 ban.url ".*"
