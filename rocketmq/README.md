#### 单机版集成控制台
#### stand-alone integrated console


**第一步**
```sh

docker pull massdock/rocketmq:5.1.0

```


**第二步**
```sh

docker run -d --name rocketmq -v D:/data/docker/rocketmq/store:/home/rocketmq/store \
    -v D:/data/docker/rocketmq/logs:/home/rocketmq/logs -v D:/data/docker/rocketmq/conf:/home/rocketmq/conf \
    -p 10911:10911 -p 10912:10912 -p 10909:10909 -p 9876:9876 -p 8181:8181 \
    massdock/rocketmq:5.1.0

```

> ps: 8181是控制台映射端口 其他都是原生rocketmq端口


**第三步**
> 须进入容器修改broker注册的IP，这样你在容器外就可以访问broker了，因为容器部署不同与Linux本地直接部署
> 一般Linux运行docker都会在宿主机注册172.17.0.1的网卡，Linux可以使用此IP，但window的docker是运行在wsl系统之上，所以window的伙伴必须修改才可以愉快的开发
> 另一种方式，使用自己的配置文件用-v进行映射

```sh
docker exec -it rocketmq /bin/bash

vi conf/broker.conf

#修改conf/broker.conf配置文件
- brokerIP1 = 172.17.0.1
+ brokerIP1 = ${你自己的宿主机IP}

# 比如我使用的window，宿主机的docker运行的wsl系统网卡IP是172.23.48.1
brokerIP1 = 172.23.48.1

# 退出容器
exit
# 再重启rocketmq容器
docker restart rockermq

```

> 使用映射的方式, 首先编辑D:/data/docker/rocketmq/conf/broker.conf, 默认已添加了mqtt的支持参数
```sh
brokerClusterName = DefaultCluster
brokerName = broker-a
brokerId = 0
brokerIP1 = 172.23.48.1
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
enableLmq = true
enableMultiDispatch = true
```

> 运行容器
```sh
docker run -d --name rocketmq -v D:/data/docker/rocketmq/store:/home/rocketmq/store \
    -v D:/data/docker/rocketmq/logs:/home/rocketmq/logs -v D:/data/docker/rocketmq/conf:/home/rocketmq/conf \
    -v D:/data/docker/rocketmq/conf/broker.conf:/home/rocketmq/rockermq-5.1.0/conf/broker.conf \
    -p 10911:10911 -p 10912:10912 -p 10909:10909 -p 9876:9876 -p 8181:8181 \
    massdock/rocketmq:5.1.0
```

至此可以在宿主机本地打开`localhost:8181`浏览控制台了
[控制台](http://localhost:8181)


#### 启动集群

> 启动nameserver

```sh
docker run -d --name mqnamesrv-0 -v D:/data/docker/rocketmq/mqnamesrv-0/store:/home/rocketmq/store \
    -v D:/data/docker/rocketmq/mqnamesrv-0/logs:/home/rocketmq/logs -v D:/data/docker/rocketmq/mqnamesrv-0/conf:/home/rocketmq/conf \
    -p 9876:9876 \
    massdock/rocketmq:5.1.0 sh bin/mqnamesrv 
```


> 启动mqbroker-a
> 注意-c指定的broker配置修改成宿主机的IP，添加 brokerIP1 = 172.23.48.1 参数

```sh
docker run -d --name mqbroker-a --link mqnamesrv-0:mqnamesrv -v D:/data/docker/rocketmq/mqbroker-a/store:/home/rocketmq/store \
    -v D:/data/docker/rocketmq/mqbroker-a/logs:/home/rocketmq/logs -v D:/data/docker/rocketmq/mqbroker-a/conf:/home/rocketmq/conf \
    -p 10911:10911 -p 10912:10912 -p 10909:10909 \
    massdock/rocketmq:5.1.0 sh bin/mqbroker -n mqnamesrv:9876 -c conf/2m-noslave/broker-a.properties --enable-proxy
```

> 启动mqbroker-b

```sh
docker run -d --name mqbroker-b --link mqnamesrv-0:mqnamesrv -v D:/data/docker/rocketmq/mqbroker-b/store:/home/rocketmq/store \
    -v D:/data/docker/rocketmq/mqbroker-b/logs:/home/rocketmq/logs -v D:/data/docker/rocketmq/mqbroker-b/conf:/home/rocketmq/conf \
    -p 11911:10911 -p 11912:10912 -p 11909:10909 \
    massdock/rocketmq:5.1.0 sh bin/mqbroker -n mqnamesrv:9876 -c conf/2m-noslave/broker-b.properties --enable-proxy
```


> 启动console控制台

```sh
docker run -d --name console -p 8181:8181 \
    massdock/rocketmq:5.1.0 java -jar rocket-dashboard.jar
```
> PS: 控制台启动后进入控制台添加 mqnamesrv, 比如我的是 172.23.48.1:9876

#### 更多参数参考官方
[Apache Rocketmq](https://rocketmq.apache.org/zh/docs/bestPractice/01bestpractice)
