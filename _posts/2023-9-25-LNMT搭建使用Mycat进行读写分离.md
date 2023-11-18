### 架构图

![架构图](https://picss.sunbangyan.cn/2023/11/14/712967c1d1f73a270dd89ef777846d4b.png)

### 环境配置

| host             | ip             |
| :--------------- | -------------- |
| nginx            | 192.168.25.150 |
| tomcat+maven     | 192.168.25.151 |
| myact+mysqlslave | 192.168.25.152 |
| mysqlmaster      | 192.168.25.153 |

使用到的源码包，上传到对应主机的/tmp目录下

**jdk**：jdk-8u201-linux-x64.tar.gz

下载：https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html

别人的地址(速度快)：https://mirrors.huaweicloud.com/java/jdk/8u201-b09/jdk-8u201-linux-x64.tar.gz

**mycat**：Mycat-server-1.6.5-release-20180122220033-linux.tar.gz

下载：http://dl.mycat.org.cn/1.6.5/Mycat-server-1.6.5-release-20180122220033-linux.tar.gz

**tomcat**：apache-tomcat-8.5.93.tar.gz

下载：https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.93/bin/apache-tomcat-8.5.93.tar.gz

**maven**：apache-maven-3.9.4-bin.tar.gz

下载：http://archive.apache.org/dist/maven/maven-3/3.9.4/binaries/apache-maven-3.9.4-bin.tar.gz

### nginx安装与配置

nginx安装

~~~shell
#配置nginx源
cat > /etc/yum.repos.d/nginx.repo << EOF
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
EOF
#安装nginx
yum install -y nginx
~~~

#### nginx配置

##### nginx.conf

~~~shell
#nginx主配置文件
cat > /etc/nginx/nginx.conf << EOF
user  nginx;
worker_processes  auto;
error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    use epoll;
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    tcp_nopush     on;
    keepalive_timeout  65;
    gzip  on;
    include /etc/nginx/conf.d/*.conf;

    upstream tomcatServer {
       server 192.168.25.151:8080 weight=1  max_fails=3  fail_timeout=10;
    }
}
~~~

##### default.conf

~~~shell
#nginx子配置文件
cat > /etc/nginx/conf.d/test.conf << EOF
proxy_cache_path /etc/nginx/zrlog_cache levels=1:2 keys_zone=zrlog_cache:100m max_size=10g inactive=1d use_temp_path=off;
proxy_headers_hash_max_size  2048;
proxy_headers_hash_bucket_size  512;

server {
    listen      8888;
    server_name  www.test.com;
    access_log  /var/log/nginx/zrlog.access.log  main;

    charset utf-8;

    proxy_buffering on;
    proxy_buffer_size  128k;
    proxy_buffers      32  16k;
    proxy_send_timeout  30s;
    proxy_read_timeout  30s;
    proxy_connect_timeout   5s;

    location / {
        proxy_pass   http://tomcatServer;
        proxy_set_header Host    $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location ~ \.(html|htm)$ {
        proxy_pass   http://tomcatServer;
    }
}
EOF
~~~

~~~shell
#设置开机自启并启动nginx
systemctl enable --now nginx
~~~



### jdk配置

分别在tomcat和mycat主机上配置jdk

~~~shell
#解压jdk
tar -xf /tmp/jdk-8u201-linux-x64.tar.gz -C /usr/local/
#给jdk文件更名
mv /usr/local/jdk1.8.0_201 /usr/local/jdk1.8
#jdk环境配置
cat > /etc/profile.d/jdk.sh << EOF
JAVA_HOME=/usr/local/jdk1.8
JRE_HOME=$JAVA_HOME/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME CLASSPATH
EOF
#使jdk环境变量生效
. /etc/profile
~~~



### tomcat安装与配置

~~~shell
#解压tomcat
tar -xf /tmp/apache-tomcat-8.5.93.tar.gz -C /opt/
#给tomcat文件名更名
mv /opt/apache-tomcat-8.5.93 /opt/tomcat-8.5
#编写一个tomcat命令，方便启动，重启和关闭
cat > /usr/bin/tomcat << EOF
#! /bin/bash
if [ $# -ne 1 ]
then
    echo "use error.  use : tomcat  start/stop/restart" >&2;
    exit 1
fi
case $1 in
    start)
        /opt/tomcat-8.5/bin/startup.sh
        ;;
    stop)
        /opt/tomcat-8.5/bin/shutdown.sh 
        ;;
    restart)
        /opt/tomcat-8.5/bin/shutdown.sh
        sleep  0.2
        /opt/tomcat-8.5/bin/startup.sh
        ;;
    *)
        echo   "tomcat  start/stop/restart"
        exit 2
        ;;
esac
exit 0
EOF
#给tomcat命令执行权限
chmod a+x /usr/bin/tomcat
~~~

~~~shell
#启动tomcat
tomcat start
~~~



### 关于war包

一般由开发者提供war包

有整个源代码也可以自行打包成war包,使用maven打包

#### maven安装与配置

##### maven安装

这里我安装在tomcat机器上，一方面是要jdk环境，另一方面是方便代码发布

~~~shell
#解压maven
tar -xf /tmp/apache-maven-3.9.4-bin.tar.gz -C /usr/local
#给maven更名
mv /usr/local/apache-maven-3.9.4 /usr/local/maven
#编写maven环境配置
cat > /etc/profile.d/maven.sh << EOF
#! /bin/bash

export MAVEN_HOME=/usr/local/maven
export PATH=$MAVEN_HOME/bin:$PATH
EOF
#使maven环境变量生效
source /etc/profile
~~~

##### maven配置

~~~shell
#修改镜像
vim /usr/local/maven/conf/settings.xml 
#源镜像在文件约160行
#注释掉或删掉原来的镜像并改成下面阿里的镜像
<mirror>
    <id>aliyunmaven</id>
    <mirrorOf>*</mirrorOf>
    <name>阿里云公共仓库</name>
    <url>https://maven.aliyun.com/repository/public</url>
</mirror>
~~~

java项目连接数据库文件 application.properties

这里指定mycat的工作ip:8066，数据库为mycat逻辑库

~~~properties
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://192.168.25.152:8066/TESTDB?characterEncoding=UTF-8
spring.datasource.username=root
spring.datasource.password=Ckc_123123

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
~~~

修改pom.xml, 添加打包方式为war包,即：

<packaging>war</packaging>

然后**在有pom.xml文件**的地方 进行maven打war包

~~~shell
mvn clean package -DskipTests
~~~

把war包丢到tomcat的webapps目录下，tomcat会自动进行解压

我这里的java项目名字叫 myweb

~~~shell
#移除tomcat原本的主页
mv /opt/tomcat-8.5/webapps/ROOT /tmp
#把打包好的war丢到tomcat的webapps目录下
cp /tmp/myweb/target/myweb-0.0.1-SNAPSHOT.war /opt/tomcat-8.5/webapps/ROOT.war
~~~



### mycat安装与配置

#### mycat安装

~~~shell
#解压mycat源码包
tar -xf /tmp/Mycat-server-1.6.5-release-20180122220033-linux.tar.gz -C /usr/local/
#创建mycat组
groupadd mycat
#创建mycat用户
useradd -g mycat mycat 
#设置mycat用户密码
echo "Ckc_123123" | passwd --stdin mycat
#更改mycat目录的权限
chown -Rf mycat:mycat /usr/local/mycat
#编写mycat的systemd来管理mycat
cat > /usr/lib/systemd/system/mycat.service << EOF
[Unit]
Description=mycat Database Proxy
Description=mycat
After=syslog.target
After=network.target

[Service]
Type=simple
Restart=on-abort
PIDFile=/usr/local/mycat/logs/mycat.pid
ExecStart=/usr/local/mycat/bin/mycat start
PrivateTmp=true

[Install]
WantedBy=multi-user.target
EOF
#修改mycat的wrapper.cnf里的java命令，设置成据对路径，可能会更稳定吧 
sed -e '5s#wrapper.java.command=java#wrapper.java.command=/usr/local/jdk1.8/bin/java#' /usr/local/mycat/conf/wrapper.conf
~~~

~~~shell
#使编写mycat的systemd文件生效
systemctl daemon-reload
#查看mycat状态，检查是否可以使用systemd进行管理
systemctl status mycat.service
~~~

#### mycat配置

##### server.xml

~~~shell
#mycat的server.xml文件
cat > /usr/local/mycat/conf/server.xml << EOF
<?xml version="1.0" encoding="UTF-8"?>
<!-- - - Licensed under the Apache License, Version 2.0 (the "License");
	- you may not use this file except in compliance with the License. - You
	may obtain a copy of the License at - - http://www.apache.org/licenses/LICENSE-2.0
	- - Unless required by applicable law or agreed to in writing, software -
	distributed under the License is distributed on an "AS IS" BASIS, - WITHOUT
	WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. - See the
	License for the specific language governing permissions and - limitations
	under the License. -->
<!DOCTYPE mycat:server SYSTEM "server.dtd">
<mycat:server xmlns:mycat="http://io.mycat/">
	<system>
	<property name="nonePasswordLogin">0</property> <!-- 0为需要密码登陆、1为不需要密码登陆 ,默认为0，设置为1则需要指定默认账户-->
	<property name="useHandshakeV10">1</property>
	<property name="useSqlStat">0</property>  <!-- 1为开启实时统计、0为关闭 -->
	<property name="useGlobleTableCheck">0</property>  <!-- 1为开启全加班一致性检测、0为关闭 -->

		<property name="sequnceHandlerType">2</property>
	<property name="subqueryRelationshipCheck">false</property> <!-- 子查询中存在关联查询的情况下,检查关联字段中是否有分片字段 .默认 false -->
      <!--  <property name="useCompression">1</property>--> <!--1为开启mysql压缩协议-->
        <!--  <property name="fakeMySQLVersion">5.6.20</property>--> <!--设置模拟的MySQL版本号-->
	<!-- <property name="processorBufferChunk">40960</property> -->
	<!--
	<property name="processors">1</property>
	<property name="processorExecutor">32</property>
	 -->
        <!--默认为type 0: DirectByteBufferPool | type 1 ByteBufferArena| type 2 NettyBufferPool -->
		<property name="processorBufferPoolType">0</property>
		<!--默认是65535 64K 用于sql解析时最大文本长度 -->
		<!--<property name="maxStringLiteralLength">65535</property>-->
		<!--<property name="sequnceHandlerType">0</property>-->
		<!--<property name="backSocketNoDelay">1</property>-->
		<!--<property name="frontSocketNoDelay">1</property>-->
		<!--<property name="processorExecutor">16</property>-->
		<!--
			<property name="serverPort">8066</property> <property name="managerPort">9066</property>
			<property name="idleTimeout">300000</property> <property name="bindIp">0.0.0.0</property>
			<property name="frontWriteQueueSize">4096</property> <property name="processors">32</property> -->
		<!--分布式事务开关，0为不过滤分布式事务，1为过滤分布式事务（如果分布式事务内只涉及全局表，则不过滤），2为不过滤分布式事务,但是记录分布式事务日志-->
		<property name="handleDistributedTransactions">0</property>

			<!--
			off heap for merge/order/group/limit      1开启  0关闭
		-->
		<property name="useOffHeapForMerge">1</property>

		<!--
			单位为m
		-->
        <property name="memoryPageSize">64k</property>

		<!--
			单位为k
		-->
		<property name="spillsFileBufferSize">1k</property>

		<property name="useStreamOutput">0</property>

		<!--
			单位为m
		-->
		<property name="systemReserveMemorySize">384m</property>


		<!--是否采用zookeeper协调切换  -->
		<property name="useZKSwitch">false</property>

		<!-- XA Recovery Log日志路径 -->
		<!--<property name="XARecoveryLogBaseDir">./</property>-->

		<!-- XA Recovery Log日志名称 -->
		<!--<property name="XARecoveryLogBaseName">tmlog</property>-->

	</system>

	<!-- 全局SQL防火墙设置 -->
	<!--白名单可以使用通配符%或着*-->
	<!--例如<host host="127.0.0.*" user="root"/>-->
	<!--例如<host host="127.0.*" user="root"/>-->
	<!--例如<host host="127.*" user="root"/>-->
	<!--例如<host host="1*7.*" user="root"/>-->
	<!--这些配置情况下对于127.0.0.1都能以root账户登录-->
	<!--
	<firewall>
	   <whitehost>
	      <host host="1*7.0.0.*" user="root"/>
	   </whitehost>
       <blacklist check="false">
       </blacklist>
	</firewall>
	-->
    <!--主要是这里的user部分，设定好密码和schema的逻辑库，逻辑库要与schema.xml的逻辑库对应-->
	<user name="root" defaultAccount="true">
		<property name="password">Ckc_123123</property>
		<property name="schemas">TESTDB</property>

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
    <!--这里我不要监控用户，所以注释了-->
		<!--
	<user name="user">
		<property name="password">user</property>
		<property name="schemas">TESTDB</property>
		<property name="readOnly">true</property>
	</user>
		 -->
</mycat:server>
EOF
~~~

##### schema.xml

~~~shell
#mycat的schema.xml文件
cat > /opt/mycat/conf/schema.xml << EOF
<?xml version="1.0"?>
<!DOCTYPE mycat:schema SYSTEM "schema.dtd">
<mycat:schema xmlns:mycat="http://io.mycat/">
        <!--name为逻辑库，checkSQLschema不进行检查，没有进行分库分表就指定好dataNode-->
        <schema name="TESTDB" checkSQLschema="false" sqlMaxLimit="100" dataNode="dn153"></schema>
        <!--name为schema的dataNode名字，database为真实存在的库-->
        <dataNode name="dn153" dataHost="192.168.25.153" database="testbookdb" />
        <!--这里要注意balance的4种用法和swithType的4种用法-->
        <dataHost name="192.168.25.153" maxCon="1000" minCon="10" balance="3"
                          writeType="0" dbType="mysql" dbDriver="native" switchType="1"  slaveThreshold="100">
                <!--心跳检查-->
                <heartbeat>select user()</heartbeat>
                <!--写操作由mysqlmaster执行-->
                <writeHost host="192.168.25.153" url="192.168.25.153:3306" user="root"
                                   password="Ckc_123123">
                        <!--读操作由mysqlslave执行-->
                        <readHost host="192.168.25.152" url="192.168.25.152:3306" user="root" password="Ckc_123123" />
                </writeHost>
        </dataHost>
</mycat:schema>
EOF
~~~

balance指的负载均衡类型，目前的取值有4种：

1. balance="0", 不开启读写分离机制，所有读操作都发送到当前可用的writeHost上。

2. balance="1"，全部的readHost与stand by writeHost参与select语句的负载均衡，简单的说，当双主双从模式(M1->S1，M2->S2，并且M1与 M2互为主备)，正常情况下，M2,S1,S2都参与select语句的负载均衡。

3. balance="2"，所有读操作都随机的在writeHost、readhost上分发。

4. balance="3"，所有读请求随机的分发到wiriterHost对应的readhost执行，writerHost不负担读压力

switchType指的是切换的模式，目前的取值也有4种：

1. switchType='-1' 表示不自动切换

2. switchType='1' 默认值，表示自动切换

3. switchType='2' 基于MySQL主从同步的状态决定是否切换,心跳语句为 show slave status

4. switchType='3'基于MySQL galary cluster的切换机制（适合集群）（1.4.1），心跳语句为 show status like 'wsrep%'。

##### log4j2.xml

~~~shell
#mycat的log4j2.xml
cat > /opt/mycat/conf/log4j2.xml << EOF
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="WARN">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d [%-5p][%t] %m %throwable{full} (%C:%F:%L) %n"/>
        </Console>

        <RollingFile name="RollingFile" fileName="${sys:MYCAT_HOME}/logs/mycat.log"
                     filePattern="${sys:MYCAT_HOME}/logs/$${date:yyyy-MM}/mycat-%d{MM-dd}-%i.log.gz">
        <PatternLayout>
                <Pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} %5p [%t] (%l) - %m%n</Pattern>
            </PatternLayout>
            <Policies>
                <OnStartupTriggeringPolicy/>
                <SizeBasedTriggeringPolicy size="250 MB"/>
                <TimeBasedTriggeringPolicy/>
            </Policies>
        </RollingFile>
    </Appenders>
    <Loggers>
        <!--<AsyncLogger name="io.mycat" level="info" includeLocation="true" additivity="false">-->
            <!--<AppenderRef ref="Console"/>-->
            <!--<AppenderRef ref="RollingFile"/>-->
        <!--</AsyncLogger>-->
        <!--把level改成debug，可以通过日志看到更多的信息-->
        <asyncRoot level="debug" includeLocation="true">

            <!--<AppenderRef ref="Console" />-->
            <AppenderRef ref="RollingFile"/>

        </asyncRoot>
    </Loggers>
</Configuration>
EOF
~~~



### MySQL安装与主从配置

#### mysql安装

~~~shell
#官网的yum源，我这里直接通过rpm安装了
rpm -ivh https://dev.mysql.com/get/mysql80-community-release-el7-11.noarch.rpm
#去掉mysql8.0的repo
sed -i '5s/0/1/g' /etc/yum.repos.d/mysql-community.repo
#开启mysql5.7的repo
sed -i '14s/1/0/g' /etc/yum.repos.d/mysql-community.repo
#安装mysql5.7
yum install -y mysql-community-server mysql-community-libs-compat
#设置跳过解析，加快MySQL的速度
echo "skip-name-resolve" >> /etc/my.cnf
#跳过权限表的，方便等下第一次登录
echo "skip-grant-tables" >> /etc/my.cnf
#重启MySQL使配置文件生效
systemctl restart mysqld
#获取MySQL的初始密码
PASSWORD=`grep 'temporary password' /var/log/mysqld.log |awk '{print $11}'`
#登录MySQL并修改成自己的密码
mysql -uroot -p"`echo $PASSWORD`" --connect-expired-password -e "flush privileges;ALTER USER 'root'@'localhost' identified by 'Ckc_123123';flush privileges;"
#去掉配置文件中跳过权限表
sed -i '$d' /etc/my.cnf
#重启MySQL使配置文件生效
systemctl restart mysqld
#删除mysql的域名源
rm -rf /etc/yum.repo.d/mysql*
#使用更改好的密码登录MySQL
mysql -uroot -pCkc_123123
~~~

#### mysql主从

##### master

~~~shell
#添加master的配置
cat >> /etc/my.cnf << EOF

log-bin=mysql-bin-master
server-id=1
binlog-ignore-db=mysql
lower_case_table_names=1
EOF
#重启MySQL使配置文件生效
systemctl restart mysqld
#给slave用户授权
mysql -uroot -pCkc_123123 -e "grant replication slave on *.* to slave@'%' identified by 'Ckc_123123';flush privileges;";
~~~

##### slave

~~~shell
#添加slave的配置
cat >> /etc/my.cnf << EOF

server-id=2
lower_case_table_names=1
EOF
#重启MySQL使配置文件生效
systemctl restart mysqld
#主从搭建
mysql -uroot -pCkc_123123 -e "stop slave;change master to master_host='192.168.25.153',master_user='slave',master_password='Ckc_123123';start slave;"
#查看主从是否搭建成功
mysql -uroot -pCkc_123123 -e "show slave status\G"
~~~



使用浏览器访问nginx配置的ip:port就可以看到java项目搭建成功