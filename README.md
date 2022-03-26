# proxy
zabbix 作为一个分布式监控系统(分布式监控解决方案)，支持通过代理(proxy)收集zabbix agent的监控数据然后由zabbix proxy再把数据发送给zabbix server，也就是zabbix proxy 可以代替 zabbix server收集监控数据，然后把数据汇报给 zabbix server，所以zabbix proxy可以在一定程度上分担了zabbix server 的数据收集压力，从而降低了数据的采集时间、也相应的增加了zabbix server的监控能力。

另外zabbix proxy也区分主动模式和被动模式，通信方式与zabbix server主动模式和被动模式一样，区别是zabbix proxy由于没有zabbix agent的配置，所以zabbix proxy在主动模式下要向zabbix server周期性申请获取zabbix agent的监控项信息，但是zabbix proxy在被动模式下也是等待zabbix server的连接并接受zabbix server发送的监控项指令，然后再由zabbix proxy向zabbix agent发起请求获取数据。

一个zabbix server可以有多个proxy，并且proxy也需要把数据临时存放在数据库中，每个proxy都有一个独立的数据库，proxy的大版本号需要与server保持一致；
apt安装proxy
1、wget https://repo.zabbix.com/zabbix/4.0/ubuntu/pool/main/z/zabbix-release/zabbix-release_4.0-3+bionic_all.deb

2、dpkg -i zabbix-release_4.0-3+bionic_all.deb

3、apt update

4、apt install zabbix-proxy-mysql -y

5、cd /usr/share/doc/zabbix-proxy-mysql/

6、zcat schema.sql.gz | mysql -uzabbix_active -plinux -h192.168.3.203 zabbix_active
#把sql语句的压缩文件通过zcat输出到终端上，通过管道导入到zabbix_active库中，进行数据库表结构的初始化
编译安装proxy
1、cd /usr/local/src

2、tar xvf zabbix-4.0.18.tar.gz

3、cd zabbix-4.0.18/

4、apt install libmysqld-dev libmysqlclient-dev libxml2-dev libxml2 snmp libsnmp-dev libevent-dev curl libcurl4-openssl-dev openjdk-8-jdk -y   #安装yilde包

5、./configure --prefix=/apps/zabbix-proxy --enable-proxy --enable-agent --enable-java --with-mysql --with-net-snmp --with-libcurl --with-libxml2
#编译安装agent，编译安装proxy，编译安装java gateway，编译安装java gateway之前需要安装jdk环境，否则环境检查时会报错；使用mysql、开启snmp、开启监控url

6、make && make install

数据库服务器：
1、mysql> create database zabbix_passive character set utf8 collate utf8_bin;
2、mysql> grant all privileges on zabbix_passive.* to zabbix_passive@'192.168.3.%' identified by 'linux';
#proxy也需要有自己的数据库

7、useradd zabbix

8、chown zabbix.zabbix /apps/zabbix-proxy/ -R

9、cd /usr/local/src/zabbix-4.0.18/database/mysql/

10、mysql -uzabbix_passive -plinux -h192.168.3.203 zabbix_passive < schema.sql 
#指定数据库，导入sql脚本，初始化proxy数据库的表结构，生成表；如果不导入sql脚本，启动proxy时会报没有表，查询不到

11、systemctl start zabbix-proxy

12、systemctl enable zabbix-proxy
准备proxy的service文件
root@test:~# vim /lib/systemd/system/zabbix-proxy.service
[Unit]
Description=Zabbix Proxy
After=syslog.target
After=network.target

[Service]
Environment="CONFFILE=/apps/zabbix-proxy/etc/zabbix_proxy.conf"
EnvironmentFile=-/etc/default/zabbix-proxy
Type=forking
Restart=on-failure
PIDFile=/tmp/zabbix_proxy.pid   #pid文件路径需要和proxy配置文件中的pid路径一致
KillMode=control-group
ExecStart=/apps/zabbix-proxy/sbin/zabbix_proxy -c $CONFFILE
ExecStop=/bin/kill -SIGTERM $MAINPID
RestartSec=10s
TimeoutSec=infinity

[Install]
WantedBy=multi-user.target
