# 分布式MySql 部署方案
---
1. 解决方案
2. 系统环境
3. mysql 主从备份
4. MyCat 中间件搭建
5. haproxy 负载代理
6. keepalived 解决单点故障
7. mycat-eye 监控web
8. 实验环境整体结构图
9. 补充

## 解决方案
### 描述
```
1. 启动mysql主从备份
2. 通过使用Mycat中间件做分表以及路由
3. 使用haproxy代理MyCat做负载均衡
4. keepalived保证haproxy的高可用性，解决单点故障。
```
### 结构图

 ![Mysql 分布式集群结构图](http://images.cnblogs.com/cnblogs_com/maybo/974565/o_mysql%E5%88%86%E5%B8%83%E5%BC%8F%E5%9B%BE.jpg)
 
##  系统环境

| system        | ip           | user  | cpu           | memory  |
| ------------- |:-------------:| -----:|------------- |:-------------:| -----:|8G|
| centos7      | 192.168.100.95 | root |cpu: Intel(R) Pentium(R) CPU G3220 @ 3.00GHz 双核|8G|
| centos7      | 192.168.100.96     |   root |cpu: Intel(R) Pentium(R) CPU G3220 @ 3.00GHz 双核|8G|
| centos7 | 192.168.100.97     |    root |cpu: Intel(R) Pentium(R) CPU G3220 @ 3.00GHz 双核|8G|

## mysql主从备份
###  修改配置文件(my.conf)

1. 主库配置

```
Server-id = 1  #这是数据库ID,此ID是唯一的，主库默认为1，其他从库以此ID进行递增，ID值不能重复，否则会同步出错；

log-bin = mysql-bin  二进制日志文件，此项为必填项，否则不能同步数据；

binlog-do-db = dbTest1  #需要同步的数据库，如果需要同步多个数据库；

则继续添加此项。

binlog-do-db = dbTest2

binlog-ignore-db = mysql 不需要同步的数据库；

```

2. 从库配置
 
```
log_bin           = mysql-bin 
server_id         = 2
relay_log         = mysql-relay-bin
log_slave_updates = 1
read_only         = 1
user            = mysql

```
### 重启数据库
### 为master数据库添加访问权限

```
create user repl;
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'192.168.100.%' IDENTIFIED BY '1234'; #访问权限添加
SHOW MASTER STATUS;  #显示主节点状态
```
### slave 备份配置
```
change master to master_host='192.168.100.96',   #master的host
master_port=3306,   #端口
master_user='repl',   #用户
master_password='1234',  #密码
master_log_file='mysql-bin.000001',   #日志文件名
master_log_pos=3204;   #开始位置将从这个位置开始备份
SHOW SLAVE STATUS;     #查看slave状态
START SLAVE;          #开启备份
STOP SLAVE;           #停止备份
 
 
 注意: 在开启备份后<SHOW SLAVE STATUS>会看到：
      Slave_IO_Runing=Yes
      Slave_SQL_Runing=Yes
      说明备份启动成功。
```
## MyCat中间件搭建
### 下载地址
<http://dl.mycat.io/1.6-RELEASE/Mycat-server-1.6-RELEASE-20161028204710-linux.tar.gz>
### 文档
<https://github.com/MyCATApache/Mycat-Server/wiki>

### 配置文件
#### server.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
        <system>
                <property name="serverPort">8066</property>
                <property name="bindIp">192.168.100.96</property>
                <property name="managerPort">9066</property>
                <property name="systemReserveMemorySize">384m</property>
                <property name="defaultSqlParser">druidparser</property>
        </system>
     <user name="admin">
                <property name="password">mypass</property>
                <property name="schemas">dbTest</property>

                <!-- 表级 DML 权限设置 -->
                <!--            
                <privileges check="false">
                        <schema name="TESTDB" dml="0110" >
                                <table name="tb01" dml="0000"></table>
                                <table name="tb02" dml="1111"></table>
                        </schema>
                </privileges>           
                 -->
        </user>

        <!--<user name="admin">
                <property name="password">mypass</property>
                <property name="schemas">db</property>
                <property name="readOnly">false</property>
        </user>-->

</mycat:server>


说明: 
    1. 结合文档很容易知道配置含义，不在说明。
    2. 主要是对外用户配置，以及管理端口，服务端口配置，和其它一些配置。
      
```
#### schema.xml
```
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">

        <schema name="dbTest" checkSQLschema="true" sqlMaxLimit="100">
                <!-- auto sharding by id (long) -->
                <table name="t_user" primaryKey="id" dataNode="dn1,dn2" rule="rule1" />

                <!-- global table is auto cloned to all defined data nodes ,so can join
                        with any table whose sharding node is in the same data node -->
                <table name="t_company" primaryKey="id" type="global" dataNode="dn1,dn2" rule="rule1" />
</schema>
        <!-- <dataNode name="dn1$0-743" dataHost="localhost1" database="db$0-743"
                /> -->
        <dataNode name="dn1" dataHost="localhost1" database="dbTest1" />
        <dataNode name="dn2" dataHost="localhost1" database="dbTest2" />
 <dataHost name="localhost1" maxCon="1000" minCon="10" balance="0"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <heartbeat>show status like 'wsrep%'</heartbeat>
                <!-- can have multi write hosts -->
                <writeHost host="hostM1" url="192.168.100.96:3306" user="admin"
                                   password="mypass">
                        <!-- can have multi read hosts -->
                        <readHost host="hostS2" url="192.168.100.97:3306" user="admin" password="mypass"/>
                </writeHost>
                <!-- <writeHost host="hostM2" url="localhost:3316" user="root" password="123456"/> -->
        </dataHost>
</mycat:schema>

说明:
     1. 数据库对应表分表配置,其中rule对应rule.xml中分表的类型。
     2. datanode 所分的数据库名字以及datahost名字。
     3.  datahost 连接配置，主数据库配置，以及从数据库配置。             
```
#### rule.xml
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mycat:rule SYSTEM "rule.dtd">
<mycat:rule xmlns:mycat="http://io.mycat/">
        <tableRule name="rule1">
                <rule>
                        <columns>id</columns>
                        <algorithm>func1</algorithm>
                </rule>
        </tableRule>
 <function name="func1" class="io.mycat.route.function.PartitionByLong">
                <property name="partitionCount">8</property>
                <property name="partitionLength">128</property>
 </function>
</mycat:rule>       

说明:
     1. 默认分表规则有很多种，可以酌情选择使用。
```
### 启动
```
1. ./bin/mycat start    #启动mycat
2. tail -n1000 -f ./logs/wrapper.log  #查看启动日志
3. tail -n1000 -f ./logs/mycat.log    #查看mycat.log服务日志
```
## haproxy 负载代理
### 下载地址
<http://www.haproxy.org/download/1.7/src/haproxy-1.7.3.tar.gz>
### 参考文档
<http://blog.csdn.net/zzhongcy/article/details/46443765>
### 安装
```
　uname -a    //查看Linux内核版本, TARGET是内核版本，2.6就写作26
　　make TARGET=linux26 PREFIX=/usr/local/haproxy
　　make install PREFIX=/usr/local/haproxy
　　
```
### 配置
 ```
 
 
 1. mkdir /etc/haproxy/conf
 2. vim /etc/haproxy/conf/haproxy.cfg


 global
log 127.0.0.1   local0 ##记日志的功能
maxconn 4096
chroot /usr/local/haproxy
user haproxy
group haproxy
daemon
########默认配置############  
defaults
log    global
mode tcp               #默认的模式mode { tcp|http|health }，tcp是4层，http是7层，health只会返回OK  
retries 3              #两次连接失败就认为是服务器不可用，也可以通过后面设置  
option redispatch      #当serverId对应的服务器挂掉后，强制定向到其他健康的服务器  
option abortonclose    #当服务器负载很高的时候，自动结束掉当前队列处理比较久的链接  
maxconn 32000          #默认的最大连接数  
timeout connect 5000ms #连接超时  
timeout client 30000ms #客户端超时  
timeout server 30000ms #服务器超时  
timeout check 2000    #心跳检测超时  
#log 127.0.0.1 local0 err #[err warning info debug]  

########test1配置#################  
listen mycat_1
bind 0.0.0.0:8076
mode tcp
balance roundrobin
server s1 192.168.100.95:8066 weight 1 maxconn 10000 check inter 10s
server s2 192.168.100.96:8066 weight 1 maxconn 10000 check inter 10s
listen mycat_1_manage
bind 0.0.0.0:9076
mode tcp
balance roundrobin
server s1 192.168.100.95:9066 weight 1 maxconn 10000 check inter 10s
server s2 192.168.100.96:9066 weight 1 maxconn 10000 check inter 10s

 ```
 ### 启动
 ```
 /usr/local/haproxy/sbin/haproxy -f /usr/local/haproxy/haproxy.cfg
 ```
 
## keepalived 解决单点故障
### 下载地址
<http://www.keepalived.org/software/keepalived-1.3.5.tar.gz> 
###  文档
<http://www.keepalived.org/documentation.html>
### 安装
```
./configure && make
```
### 配置
```
1. mkdir -p  /usr/local/etc/keepalived/
2. vim /usr/local/etc/keepalived/keepalived.conf

global_defs {
    router_id NodeB
}
vrrp_instance VI_1 {
    state BACKUP    #设置为主服务器
    interface enp3s0  #监测网络接口
    virtual_router_id 51  #主、备必须一样
    priority 90   #(主、备机取不同的优先级，主机值较大，备份机值较小,值越大优先级越高)
    advert_int 1   #VRRP Multicast广播周期秒数
    authentication {
    auth_type PASS  #VRRP认证方式，主备必须一致
    auth_pass 1111   #(密码)
}
virtual_ipaddress {
    192.168.200.100/24  #VRRP HA虚拟地址
}

```
### 启动
```
./bin/keepalived -D -f /usr/local/etc/keepalived/keepalived.conf
```
## mycat-eye 监控web
### 下载地址
```
http://dl.mycat.io/mycat-web-1.0/Mycat-web-1.0-SNAPSHOT-20170102153329-linux.tar.gz
```
###  安装zookeeper
```
docker run -d \
-e MYID=1 \
--name=zookeeper --net=host --restart=always sdvdxl/zookeeper
```
### 配置
```
修改zookeeper地址： 
       1. cd   /mycat-web/WEB-INF/classes 
       2. vim mycat.properties 
       3. zookeeper=127.0.0.1:2181 
```
### 启动
```
1. cd /mycat-web/ 
2. ./start.sh & 
```
## 实验环境整体结构图
![mysq分布式整体架构图](http://images.cnblogs.com/cnblogs_com/maybo/974565/o_mysql%e5%88%86%e5%b8%83%e5%bc%8f%e5%ae%9e%e9%aa%8c%e6%9e%b6%e6%9e%84%e5%9b%be.jpg)
## 补充
###  MyCat 密码明文加密

```
  1. java -cp Mycat-server-1.6-RELEASE.jar io.mycat.util.DecryptUtil   1:userB:root:321
  2. 修改配置 <property name="usingDecrypt">1</property> #使用加密
  
  说明:
     1. 0 为对外提供密码加密，1.是后端也就是数据库连接密码加密
     2. userB 用户名
     3. 321 明文密码
```

