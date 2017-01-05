#!/bin/bash
#LANG=en_US.UTF-8
#      exit code
# exit 39 yum error
# exit 40 configure error
# exit 41 make error
# exit 42 make install errot

#static variable
INSTALL_FILE="varnish-3.0.6.tar.gz"
CODE_DIR=$(tar -tf $INSTALL_FILE | head -1)
NULL=/dev/null

test_yum () {
#set yum configure file do not display Red Hat Subscription Management info.
	if [ -f /etc/yum/pluginconf.d/subscription-manager.conf ];then
		sed -i '/enabled/s/1/0/' /etc/yum/pluginconf.d/subscription-manager.conf
	fi
	yum clean all &>$NULL
	repolist=$(yum repolist 2>/dev/null |awk '/repolist:/{print $2}'|sed 's/,//')
	if [ $repolist -le 0 ];then
		exit 39
	fi
}

test_yum
##安装variable需要依赖包
yum -y install gcc* readline-devel pcre-devel &> $NULL

##添加对应用户，以该用户进行操作后续
useradd -s /sbin/nologin varnish

tar -xf $INSTALL_FILE
cd $CODE_DIR
[ -f configure ] && ./configure --prefix=/usr/local/varnish  &> $NULL || exit 40
echo "configure success."
make &> $NULL
[ $? -eq 0 ] && echo "make success."  || exit 41
make install &> $NULL
[ $? -eq 0 ] && echo "make install success." || exit 42
##进行配置文件的复制操作；
cp redhat/varnish.initrc /etc/init.d/varnish
cp redhat/varnish.sysconfig  /etc/sysconfig/varnish
cp redhat/varnish_reload_vcl /usr/bin/
ln -s /usr/local/varnish/sbin/varnishd /usr/sbin/
mkdir /etc/varnish
cp /usr/local/varnish/etc/varnish/default.vcl /etc/varnish/
uuidgen > /etc/varnish/secret

#最大的线程数和最小线程数受计算机的配置有关，其中什么时候增加进程，在有些配置文件会添加一个空
#闲线程的参数，通过该参数进行相关的操作
#VARNISH_ADMIN_LISTEN_ADDRESS 最小线程数
#VARNISH_MAX_THREADS=1000 最大线程数
#VARNISH_STORAGE_SIZE=64M 缓存大小,缓存大小受业务和缓存总容量影响。
#VARNISH_STORAGE="malloc,${VARNISH_STORAGE_SIZE}" 使用内存缓存页面，内存大小为64M,
#还可以使用硬盘进行缓存操作，不过那样速度会慢，无法体现出varnish的优势
# VARNISH_VCL_CONF=/etc/varnish/default.vcl  

#sed -n '/VARNISH_VCL_CONF=/p' /etc/sysconfig/varnish 
#VARNISH_LISTEN_PORT 默认端口,修改为httpd默认的端口
sed -i '/VARNISH_LISTEN_PORT=/  s/=.*/=80/' /etc/sysconfig/varnish
sed -i '/VARNISH_MIN_THREADS=/ s/=.*/=1000/' /etc/sysconfig/varnish
sed -i '/VARNISH_MAX_THREADS=/ s/=.*/=10000/' /etc/sysconfig/varnish
sed -i '/VARNISH_STORAGE_SIZE=/ s/=.*/=128M/' /etc/sysconfig/varnish
sed -i '/VARNISH_STORAGE=/ s/=.*/="malloc,${VARNISH_STORAGE_SIZE}"/' /etc/sysconfig/varnish 

#修改主配置文件（定义后台服务器）default.vcl
#backend default {
#     .host = "192.168.2.100";
#	      .port = "80";
#	   }
sed -i '/backend default/ s/.*back/back/' /etc/varnish/default.vcl
sed -i '/.host =/ s/.*\.host =.*/    .host = "192.168.2.16";/' /etc/varnish/default.vcl
sed -i '/.port =/ s/.*\.port =.*/    .port = "80";/' /etc/varnish/default.vcl
sed -i '/.port =/a }' /etc/varnish/default.vcl

#当对网页的信息更新频率要求很高时，就可以使用下列命令进行设置
#ln -s /usr/local/varnish/bin/* /usr/bin/
#varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082 ban.url index.html
#varnishadm -S /etc/varnish/secret -T 127.0.0.1:6082 ban.url ".*"


