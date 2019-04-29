# ELK(elasticsearch+logstash+kibana) 搭建

*搭建完以后敲的，有误留言，谢谢。*

------

#### 准备环境

<font color=red>***Tip: 所有操作均通过普通用户sudo执行***</font>

> 系统CentOS 7.x  JDK: 一定要用官网的 1.8

> Elasticsearch |Logstash | kibana-5.0.0

#### 环境搭建

1. JDK配置

```bash
vim /etc/profile
#JDk Config
JAVA_HOME=/opt/jdk1.8.0_111
JRE_HOME=$JAVA_HOME/jre
CLASSPATH=.:$JAVA_HOME/lib:/dt.jar:$JAVA_HOME/lib/tools.jar
PATH=$PATH:$JAVA_HOME/bin
export  JAVA_HOME
export  JRE_HOME
#Process Max
ulimit -SHn 65537
#配置生效
 source /etc/profile
```
2. ELK基础环境

```bash
#create data source
mkdir –p /opt/tmp/data
mkdir –p /opt/tmp/log
mkdir –p /opt/backup
#add user
useradd elk
groupadd elks
usermod -a –G elks elk 
usermod -a –G root elk
#解压缩安装包

```

3. ELK配置文件

*vim /opt/logstash/config/logstash.conf*

```bash
input {
	file {
		path => ["/opt/tmp/Logstash.log"]
		add_field => {
			"appName" => "elk"
		}
		type => "elk"
	}
	beats {
		port => 5044
	}
}
filter {
	grok {
		match => {
			"message" => "msg"
		}
	}
}
output {
	elasticsearch {
		hosts => ["192.168.152.128:9200"]
	}
	stdout {
		codec => rubydebug
	}
}
# end
```

*vim /opt/elist/config/elasticsearch.yml*
```bash
#写入以下内容，每个:后都有一个空格！！！
path.data: /opt/tmp/data
path.logs: /opt/tmp/log
network.host: 192.168.152.128
http.port: 9200
#为防止Elastic启动报错
http.cors.allow-origin: "/.*/"
http.cors.enabled: true
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
```

*vim /opt/elist/config/elasticsearch.yml*
```bash
#写入以下内容，每个:后都有一个空格！！！
path.data: /opt/tmp/data
path.logs: /opt/tmp/log
network.host: 192.168.152.128
http.port: 9200
#为防止Elastic启动报错
http.cors.allow-origin: "/.*/"
http.cors.enabled: true
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
```

*vim /opt/elist/config/jvm.options*

```bash
#内存大的话可以设置内存的1/4
-Xms512m
-Xmx512m
```

#### 问题列表

##### elasticsearch 无法启动

```bash
#给es目录加权限
chown –R elkselk
chown –R elk.elks /opt/
```

##### SecComp不支持

```bash
# ERROR: bootstrap checks failed
#system call filters failed to install; check the logs and fix your #configuration or disable system call filters at your own risk

#原因：这是在因为Centos6不支持SecComp，而ES5.2.0默认bootstrap.system_call_filter为true进行检测，所以导致检测失败，失败后直接导致ES不能启动。

#解决方案
vim elasticsearch.yml
#在Memory内容块下添加如下内容
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
```

##### unable to install syscall filter

```bash
#Java.lang.UnsupportedOperationException: seccomp unavailable: requires kernel 3.5+ with CONFIG_SECCOMPandCONFIG_SECCOMP_FILTERcompiledinatorg.elasticsearch.bootstrap.Seccomp.linuxImpl(Seccomp.java:349) ~[elasticsearch-5.0.0.jar:5.0.0] at org.elasticsearch.bootstrap.Seccomp.init(Seccomp.java:630) ~[elasticsearch-5.0.0.jar:5.0.0]

升级新系统
```

##### bootstrap checks failed max file descriptors

```bash
#ERROR: bootstrap checks failed max file descriptors [4096] for elasticsearch process likely too low, increase to at least [65536]
sudo vim /etc/security/limits.conf
#添加如下内容 * 代表用户
* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096
#保存后执行 ulimit -a ，注销重新登陆。
```

##### max number of threads [1024] for user [es] likely too low, increase to at least [2048]

```bash
#原因：无法创建本地线程问题,用户最大可创建线程数太小
#解决方案
vim /etc/security/limits.d/90-nproc.conf
#修改如下内容
# * soft nproc 1024
* soft nproc 2048
```

##### max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]

```bash
#原因：最大虚拟内存太小
#解决方案：
sudo vim /etc/sysctl.conf
#添加如下内容
vm.max_map_count=655360
#保存后执行sysctl -p
```

##### Elasticsearch 无plugins脚本

```bash
#ES5.X无plugins脚本，所以 5.x以上使用
./elasticsearch-plugin install x-pack
#5.x以下采用
plugins install xxx
#新版本采用如下方式可安装
rpm - ivh xxx.rpm
```

##### Elasticsearch 无法使用root用户运行

```bash
# 方式一：运行时，添加-Des.insecure.allow.root=true该参数
./elasticsearch -Des.insecure.allow.root=true
# 方式二：
#创建用户组，并添加用户至用户组：
groupadd elasticsearch
useradd elasticsearch -g elasticsearch -p password
#修改es文件夹
chown -R elasticsearch:elasticsearch elasticsearch路径
#方式三： 修改es启动文件
vim elasticsearch
#修改以下参数
ES_JAVA_OPTS="-Des.insecure.allow.root=true"  
```



# 参考资料

官网： www.elastic.co

JDK配置： http://www.linuxidc.com/Linux/2016-03/128794.htm

ELK优化：<http://www.cnblogs.com/along21/archive/2018/03/21/8613115.html>

Other技巧：<https://www.jianshu.com/p/79b8b302098d>



> 转载请注明来源于：mu0gua.github.io