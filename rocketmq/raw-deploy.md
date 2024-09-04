##### Centos Deploy Rocketmq MQTT

`````sh
# ubuntu:22.04
export DEBIAN_FRONTEND=noninteractive

apt update
apt-get install -y --no-install-recommends gnupg2 unzip curl openjdk-8-jdk net-tools iputils-ping vim git

```

## 安装 maven

````sh
mkdir -p /home/rocket/maven
cd /home/rocket/maven
curl -sSL -O https://dlcdn.apache.org/maven/maven-3/3.8.8/binaries/apache-maven-3.8.8-bin.tar.gz
tar -zxvf apache-maven-3.8.8-bin.tar.gz
mv ./apache-maven-3.8.8 /usr/local/maven
rm -f apache-maven*.gz
cd ..
rm -f /home/rocket/maven

```



## **下载安装 Rocketmq**

```sh
mkdir -p /home/rocket/rocketmq
cd  /home/rocket/rocketmq

rocket_version=5.3.0
curl -O https://dist.apache.org/repos/dist/release/rocketmq/${rocket_version}/rocketmq-all-${rocket_version}-bin-release.zip
unzip rocketmq-all-${rocket_version}-bin-release.zip -d ./
mv rocketmq-all-*/* .
rm -rf rocketmq-all-*
```

## 配置环境变量

```sh
cp /etc/profile /etc/profile.bk
cat >> /etc/profile <<EOF
export M2_HOME=/usr/local/maven
export ROCKET_HOME=/home/rocket/rocketmq
export JAVA_HOME=$(dirname $(dirname $(readlink -f $(which javac))))
export PATH=\$ROCKET_HOME/bin:\$M2_HOME/bin:\$JAVA_HOME/bin:\$PATH
EOF
source /etc/profile

```

> **修改 Brock.conf 配置文件**
>
> vi conf/broker.conf

```sh
# 指定broker可以让其他服务可以访问的broker地址
brokerIP1 = 192.168.1.2
# 启用mqtt消息分发
enableLmq = true
enableMultiDispatch = true
```

> **启动 Rocketmq**

```sh
# 启动 nameserver
nohup sh bin/mqnamesrv >~/logs/rocketmqlogs/namesrv.log 2>&1 &
# 启动 broker
nohup sh bin/mqbroker -n localhost:9876 --enable-proxy &
```


> **在 rocketmq 中创建 mqtt 对应的两个 topic 创建好备用**
>
> eventNotifyRetryTopic=eventMqttNotifyRetryTopic
> clientRetryTopic=clientMqttRetryTopic

```sh
# 创建mqtt topic
mqadmin updatetopic -c DefaultCluster -t eventMqttNotifyRetryTopic -n localhost:9876
mqadmin updatetopic -c DefaultCluster -t clientMqttRetryTopic -n localhost:9876
mqadmin updatetopic -c DefaultCluster -t markAditFirstMessageTopic -n localhost:9876
# 更新rocketKv配置mqtt提供外部可访问的路由节点IP
mqadmin updateKvConfig -s LMQ -k LMQ_CONNECT_NODES -v localhost,172.23.0.1 -n localhost:9876
# 更新rocketKv配置所有前缀主题TOPIC
mqadmin updateKvConfig -s LMQ -k ALL_FIRST_TOPICS -v eventMqttNotifyRetryTopic,clientMqttRetryTopic,markAditFirstMessageTopic -n localhost:9876
mqadmin updateKvConfig  -s LMQ -k eventMqttNotifyRetryTopic -v eventMqttNotifyRetryTopic/+  -n localhost:9876
mqadmin updateKvConfig  -s LMQ -k clientMqttRetryTopic -v clientMqttRetryTopic/+  -n localhost:9876
mqadmin updateKvConfig  -s LMQ -k markAditFirstMessageTopic -v markAditFirstMessageTopic/+  -n localhost:9876
```

## 下载安装 MQTT

> **克隆 MQTT 源码 编译**
```sh
mkdir -p /home/rocket/mqtt
cd /home/rocket/mqtt

git clone https://github.com/apache/rocketmq-mqtt.git
cd rocketmq-mqtt
mvn -Prelease-all -DskipTests clean install -U
mv distribution/target/rocketmq-mqtt-*.zip ../
cd ..
unzip rocketmq-mqtt-*.zip
rm -f rocketmq-mqtt-*.zip
mv rocketmq-mqtt-*/* .
rm -rf rocketmq-mqtt*
```

### 配置 MQTT 开始

> **修改 mqtt service.conf 文件**
>
> vi conf/service.conf

```sh
username=admin                      #注意：这里的用户名会在客户端中使用
secretKey=adminkey                 #注意：这里的用户密码会在客户端中使用
# Rocketmq的nameserver地址端口
NAMESRV_ADDR=localhost:9876
eventNotifyRetryTopic=eventMqttNotifyRetryTopic
clientRetryTopic=clientMqttRetryTopic
# Mqtt meta的地址端口
metaAddr=localhost:8561
```

> **修改 MQTT meta.conf 文件**
>
> vi conf/meta.conf

```sh
selfAddress=localhost:8561
membersAddress=localhost:8561
```

> **启动 mqtt**

```sh
# 先启动meta
sh bin/meta.sh start
sh bin/mqtt.sh start
```

> ### 接下来 MQTT 玩耍吧
`````
