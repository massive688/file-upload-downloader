##### Centos Deploy Rocketmq MQTT

~~~sh
# Centos:7.9.2009
curl -o /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo

yum update -y
yum install -y java-1.8.0-openjdk-devel.x86_64 unzip wget telnet git

cd ~

# 安装maven
curl -sSL -O https://dlcdn.apache.org/maven/maven-3/3.8.8/binaries/apache-maven-3.8.8-bin.tar.gz && \
tar -zxvf apache-maven-3.8.8-bin.tar.gz && \
mv ./apache-maven-3.8.8 /usr/local/maven && \
rm -f apache-maven*.gz
~~~



> **创建一个根目录**

~~~sh
mkdir ~/rocketmq
mkdir -p ~/logs/rocketmqlogs
cd ~/rocketmq
~~~

> **下载Rocketmq**

~~~sh
wget https://dist.apache.org/repos/dist/release/rocketmq/5.1.3/rocketmq-all-5.1.3-bin-release.zip
unzip rocketmq-all-5.1.3-bin-release.zip -d ./
mv rocketmq-all-*/* . 
~~~


~~~sh
cp /etc/profile /etc/profile.bk
cat >> /etc/profile <<EOF
export M2_HOME=/usr/local/maven
export ROCKET_HOME=/root/rocketmq
export PATH=\$ROCKET_HOME/bin:\$M2_HOME/bin:\$PATH
EOF
source /etc/profile

~~~



> **修改Brock.conf配置文件**
>
> vi conf/broker.conf

~~~sh
# 指定broker可以让其他服务可以访问的broker地址
brokerIP1 = 192.168.0.9
# 启用mqtt消息分发
enableLmq = true
enableMultiDispatch = true
~~~

> **启动 Rocketmq**

~~~sh
# 启动nameserver
nohup sh bin/mqnamesrv >~/logs/rocketmqlogs/namesrv.log 2>&1 &
# 启动broker
nohup sh bin/mqbroker -n localhost:9876 --enable-proxy &
~~~

> **在rocketmq中创建mqtt对应的两个topic 创建好备用**
>
> ```sh
> eventNotifyRetryTopic=eventMqttNotifyRetryTopic
> clientRetryTopic=clientMqttRetryTopic
> ```

```sh
# 创建mqtt topic 
mqadmin updatetopic -c DefaultCluster -t eventMqttNotifyRetryTopic -n localhost:9876
mqadmin updatetopic -c DefaultCluster -t clientMqttRetryTopic -n localhost:9876
# 更新rocketKv配置mqtt提供外部可访问的路由节点IP
mqadmin updateKvConfig -s LMQ -k LMQ_CONNECT_NODES -v localhost,192.168.0.9,172.23.0.1 -n localhost:9876
# 更新rocketKv配置所有前缀主题TOPIC 
mqadmin updateKvConfig -s LMQ -k ALL_FIRST_TOPICS -v eventMqttNotifyRetryTopic,clientMqttRetryTopic,markAditFirstMessageTopic -n localhost:9876
mqadmin updateKvConfig  -s LMQ -k eventMqttNotifyRetryTopic -v eventMqttNotifyRetryTopic/+  -n localhost:9876
mqadmin updateKvConfig  -s LMQ -k clientMqttRetryTopic -v clientMqttRetryTopic/+  -n localhost:9876
```

~~~sh
# 创建自定义mqtt TOPIC
mqadmin updatetopic -c DefaultCluster -t markAditFirstMessageTopic -n localhost:9876
mqadmin updateKvConfig -s LMQ -k ALL_FIRST_TOPICS -v eventMqttNotifyRetryTopic,clientMqttRetryTopic,markAditFirstMessageTopic -n localhost:9876
mqadmin updateKvConfig  -s LMQ -k markAditFirstMessageTopic -v markAditFirstMessageTopic/+ -n localhost:9876

# 创建自定义普通TOPIC
mqadmin updatetopic -c DefaultCluster -t maskTopic -n localhost:9876
~~~



> ### 配置MQTT开始

~~~sh
mkdir ~/mqtt
cd ~/mqtt
~~~



> **克隆MQTT源码 编译**

~~~sh
git clone https://github.com/apache/rocketmq-mqtt
cd rocketmq-mqtt 
mvn -Prelease-all -DskipTests clean install -U 
mv distribution/target/rocketmq-mqtt-*.zip ../ 
cd ..
unzip rocketmq-mqtt-*.zip
cd rocketmq-mqtt-*
~~~

> **修改mqtt service.conf文件**
>
> vi conf/service.conf

~~~sh
username=admin                     	#注意：这里的用户名会在客户端中使用
secretKey=adminkey                	#注意：这里的用户密码会在客户端中使用
# Rocketmq的nameserver地址端口
NAMESRV_ADDR=192.168.0.9:9876
eventNotifyRetryTopic=eventMqttNotifyRetryTopic
clientRetryTopic=clientMqttRetryTopic
# Mqtt meta的地址端口
metaAddr=192.168.0.9:8561

# 对应connect.conf rpcListenPort参数端口
csRpcPort=7001
~~~

> **修改MQTT meta.conf文件**
>
> vi conf/meta.conf

~~~sh
selfAddress=192.168.0.9:8561
membersAddress=192.168.0.9:8561
~~~

> **修改MQTT connect.conf文件**
>
> vi conf/connect.conf

~~~sh
rpcListenPort=7001
~~~



> **启动mqtt**

~~~sh
# 先启动meta
sh bin/meta.sh start
sh bin/mqtt.sh start
~~~

> ### 接下来MQTT玩耍吧
