> **创建一个根目录**

~~~sh
mkdir rocketmq
cd rocketmq
~~~

> **下载Rocketmq**

~~~sh
wget https://dist.apache.org/repos/dist/release/rocketmq/5.1.3/rocketmq-all-5.1.3-bin-release.zip
unzip rocketmq-all-5.1.3-bin-release.zip -d ./
mv rocketmq*/* . 
sed -i 's/\r$//' bin/*.sh
sed -i 's/\r$//' *.sh
chown -R ${uid}:${gid} ${ROCKETMQ_HOME}
~~~

> **修改Brock.conf配置文件**
>
> `vi conf/broker.conf`

~~~sh
# 指定broker可以让其他服务可以访问的broker地址
brokerIP1 = 172.23.0.1
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



> **克隆MQTT源码**

~~~sh
git clone https://github.com/apache/rocketmq-mqtt
cd rocketmq-mqtt 
~~~

> 修改mqtt service.conf文件
>
> `vi distribution/conf/service.conf`

~~~sh
username=admin
secretKey=adminKey
# Rocketmq的nameserver地址端口
NAMESRV_ADDR=172.23.0.1:9876
eventNotifyRetryTopic=eventMqttNotifyRetryTopic
clientRetryTopic=clientMqttRetryTopic
# Mqtt meta的地址端口
metaAddr=172.23.0.1:8561
~~~

> **修改MQTT meta.conf文件**
>
> `vi distribution/conf/meta.conf`

~~~sh
selfAddress=172.23.0.1:8561
membersAddress=172.23.0.1:8561
~~~

> **编译**

~~~sh
mvn -Prelease-all -DskipTests clean install -U 
cd distribution/target/ 
cd bin
~~~

> **在rocketmq中创建mqtt对应的两个topic**
>
> ```sh
> eventNotifyRetryTopic=eventMqttNotifyRetryTopic
> clientRetryTopic=clientMqttRetryTopic
> ```

```sh
# 创建mqtt topic 
sh bin/mqadmin updatetopic -c DefaultCluster -t eventMqttNotifyRetryTopic -n localhost:9876
sh bin/mqadmin updatetopic -c DefaultCluster -t clientMqttRetryTopic -n localhost:9876
# 更新rocketKv配置 
sh bin/mqadmin updateKvConfig -s LMQ -k LMQ_CONNECT_NODES -v localhost -n localhost:9876
sh bin/mqadmin updateKvConfig -s LMQ -k ALL_FIRST_TOPICS -v eventMqttNotifyRetryTopic,clientMqttRetryTopic -n localhost:9876
sh bin/mqadmin updateKvConfig  -s LMQ -k eventMqttNotifyRetryTopic -v eventMqttNotifyRetryTopic/+  -n localhost:9876
sh bin/mqadmin updateKvConfig  -s LMQ -k clientMqttRetryTopic -v clientMqttRetryTopic/+  -n localhost:9876
```
> **启动mqtt**

~~~sh
# 先启动meta
sh meta.sh start
sh mqtt.sh start
~~~

> ### 接下来MQTT玩耍吧

