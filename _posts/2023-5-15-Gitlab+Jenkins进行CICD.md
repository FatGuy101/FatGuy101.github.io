---
layout: post
title: CICD
subtitle: 使用Jenkins+Gitlab
categories: markdown
tags: [Linux]
---



### gitlab安装部署

配置依赖环境

```shell
yum install -y curl policycoreutils-python openssh-server openssh-clirnts postfixcronie lokkit rpm git
```

安装gitlab

```shell
rpm -ivh /tmp/gitlab-ce-16.0.0-ce.0.el7.x86_64.rpm
```

修改配置文件

~~~shell
vim /etc/gitlab/gitlab.rb 

external_url 'http://192.168.25.130'
user['username'] = "git"
user['group'] = "git"
~~~

修改配置文件后

~~~shell
gitlab-ctl reconfigure
~~~

成功后

~~~shell
Notes:
Default admin account has been configured with following details:
Username: root
Password: You didn't opt-in to print initial root password to STDOUT.
Password stored to /etc/gitlab/initial_root_password. This file will be cleaned up in first reconfigure run after 24 hours.

NOTE: Because these credentials might be present in your log files in plain text, it is highly recommended to reset the password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

gitlab Reconfigured!
~~~

查看初始密码

~~~shell
cat /etc/gitlab/initial_root_password

# WARNING: This value is valid only in the following conditions
#          1. If provided manually (either via `GITLAB_ROOT_PASSWORD` environment variable or via `gitlab_rails['initial_root_password']` setting in `gitlab.rb`, it was provided before database was seeded for the first time (usually, the first reconfigure run).
#          2. Password hasn't been changed manually, either via UI or via command line.
#
#          If the password shown here doesn't work, you must reset the admin password following https://docs.gitlab.com/ee/security/reset_user_password.html#reset-your-root-password.

Password: 3+ZV2iUGZGUhtVXNvt40wD3z6Fwur/RSgBKGxZ/w6vk=

# NOTE: This file will be automatically deleted in the first reconfigure run after 24 hours.

~~~

访问  http://192.168.25.130

第一次登录使用初始密码进行登录，账号为 root

进去后，修改密码，设置语言

创建用户--->创建群组--->创建项目

![](https://picss.sunbangyan.cn/2023/11/27/9ef430ff9c39fd23452ea9b63c0c7c82.jpeg)

![](https://picst.sunbangyan.cn/2023/11/27/5b0e9d6372a7945507f143daaf16bffc.jpeg)

![](https://picdl.sunbangyan.cn/2023/11/27/099c9875446ffd1a0d653771c991b7cb.jpeg)

修改ssh不进行指纹验证

~~~shell
vim /etc/ssh/ssh_config
# 约35行改成 
StrictHostKeyChecking no
~~~

允许webhook

![](https://picdl.sunbangyan.cn/2023/11/27/b2ca3a9fefd07286de0bc8195225e1c1.jpeg)





### jenkins安装部署

#### jdk17

~~~shell
cd /tmp
wget https://download.oracle.com/java/17/latest/jdk-17_linux-x64_bin.rpm
rpm -ivh jdk-17_linux-x64_bin.rpm
# java路径
# /usr/lib/jvm/jdk-17-oracle-x64/bin/java
~~~

环境变量

~~~shell
cat > /etc/profile.d/jdk.sh << EOF
#! /bin/bash
JAVA_HOME=/usr/java/jdk-17
JRE_HOME=$JAVA_HOME/jre
PATH=$PATH:$JAVA_HOME/bin:$JRE_HOME/bin
CLASSPATH=$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$JRE_HOME/lib
export JAVA_HOME JRE_HOME CLASSPATH
EOF
source /etc/profile
~~~

#### 安装maven

~~~shell
#上传并解压maven源码包
tar -xf /tmp/apache-maven-3.9.4-bin.tar.gz -C /usr/local
#更名便于操作
mv /usr/local/apache-maven-3.9.4 /usr/local/maven
~~~

配置环境变量

~~~shell
cat > /etc/profile.d/maven.sh << EOF
#! /bin/bash
export MAVEN_HOME=/usr/local/maven
export PATH=$MAVEN_HOME/bin:$PATH
EOF
~~~

~~~shell
source /etc/profile
~~~

查看maven版本

~~~~shell
mvn -v
~~~~

`Apache Maven 3.9.4 (dfbb324ad4a7c8fb0bf182e6d91b0ae20e3d2dd9)`
`Maven home: /usr/local/maven`
`Java version: 17.0.9, vendor: Oracle Corporation, runtime: /usr/lib/jvm/jdk-17-oracle-x64`
`Default locale: en_US, platform encoding: UTF-8`
`OS name: "linux", version: "3.10.0-1160.71.1.el7.x86_64", arch: "amd64", family: "unix"`

配置maven加速器

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



官网：https://pkg.jenkins.io/redhat-stable/

#### 安装jenkins

~~~shell
wget -O /etc/yum.repos.d/jenkins.repo https://pkg.jenkins.io/redhat-stable/jenkins.repo
rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key
yum install -y fontconfig java-17-openjdk jenkins git
systemctl enable --now jenkins.service
~~~

查看初始密码

~~~shell
cat /var/lib/jenkins/secrets/initialAdminPassword
~~~

系统配置

![](https://picst.sunbangyan.cn/2023/11/27/ad8d8057cf336dc32e2efae090b28bec.jpeg)

全局配置

![](https://picss.sunbangyan.cn/2023/11/27/d31f0fe44a3c4d7e7d7d72982ab7939e.jpeg)

安装插件

![](https://picss.sunbangyan.cn/2023/11/27/f721522006daa4bf5fe99195e126e180.jpeg)

获取jenkins的ssh密钥

~~~shell
usermod -s /bin/bash jenkins
su - jenkins
cat .ssh/id_rsa.pub
usermod -s /bin/false jenkins
~~~

`ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQC5G5XbbJ1QiU7qKBudyLgLKaCYJ2Y7/+o/WrjO++qUdbGbjz1cDizqXywVbt6Jm1QEES87K9isGhjSVffp36zfHOpa1RNdvgxc8QNjwVQ9M0IpTryn1UJMdOcFLpS62vpbC2jzZVc+KKdZ9Fya//HccIymB10YBQQX1yWYeYSz6+K/olDLa+qUqom1Ycs00xBShXTvzh84fdUbMh2Rr5sYhbu4sg1tqQOZsK4k+72FfkVy9qPF1HVaicxKeJ0dVR9juqOq1Tdgq0ae+CkpR//sUvPJd7RLZXo58BzYJQG8bbz0xvBjHa8IUjnmFY77ofrN82+iGkt8vDBu2jeMHIg9 jenkins@jenkins131`

获取的ssh密钥可以丢到gitlab

进去后在最下面的 jenkins中文社区修改镜像

升级站点的url填进 清华镜像

~~~
https://mirrors.tuna.tsinghua.edu.cn/jenkins/updates/update-center.json
~~~

修改一下ssh配置

~~~shell
vim /etc/ssh/ssh_config
# 约35行改成
StrictHostKeyChecking no
~~~

配置jenkins权限

~~~shell
visudo
# 约100行添加下面内容

jenkins ALL=(ALL)       NOPASSWD: ALL
~~~



### 创建pipline流水线

![](https://picst.sunbangyan.cn/2023/11/27/1a39394267c13164c78d627f5b08afca.jpeg)

配置job

![](https://picst.sunbangyan.cn/2023/11/27/7e9415f7d7b4fdd6eacd924801a97c19.jpeg)

gitlab配置webhook

![](https://picdl.sunbangyan.cn/2023/11/27/133936b1bf02f6225049532c6685b16b.jpeg)

pipeline流水线脚本

~~~pipeline
pipeline {
    agent any

    stages {
        stage('git') {
            steps {
                sleep 1
                git changelog: false, poll: false, url: 'git@192.168.25.130:myweb/myweb.git'
            }
        }
        stage('test') {
            steps {
                sh  'mvn clean package -DskipTests'
            }
        }
        stage('deploy') {
            steps {
                sh  'sudo sshpass  -p 123456 ssh  root@192.168.25.153 rm -rf /tmp/tomcatdata/*'
                sh  'sudo sshpass  -p 123456 scp  -r  /var/lib/jenkins/workspace/myweb/target/myweb-0.0.1-SNAPSHOT.war   192.168.25.153:/tmp/tomcatdata/ROOT.war'
            }
        }
    }
}
~~~

触发Gitlab推送事件，Jenkins流水线也会被触发

![](https://picdm.sunbangyan.cn/2023/11/27/96741eef8f44b7c322c71f33aefd4a57.jpeg)

