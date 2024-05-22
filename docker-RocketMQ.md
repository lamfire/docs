# 使用docker-compose部署RocketMQ 4.9.4集群

#### 1.下载镜像

```
docker pull apache/rocketmq:4.9.4
docker pull apacherocketmq/rocketmq-dashboard

```

主要用到了2个镜像，第1个用来部署NameServer和Broker，第2个用来管理rocketmq.



#### 2.部署

本文主要采用一台服务器来部署RocketMQ集群，集群部署模式采用多master模式，也就是2个NameServer和2个Broker，刷盘机制采用异步复制，异步刷盘。

##### 创建目录

```
mkdir -p /data/rocketmq/
cd /data/rocketmq/

mkdir -p nameserver/logs
mkdir -p broker/logs
mkdir -p broker/store
mkdir -p console-ng/data

```

设置权限

```
chmod 777 -R ./*
```

注意：这里如果不设置权限，会导致docker写入文件失败，导致rocketmq启动异常。



##### 创建broker.conf

```bash
vim ./broker/broker.conf

brokerClusterName = rocketmq-cluster
brokerName = broker
brokerId = 0
brokerIP1 = 192.168.1.212
deleteWhen = 04
fileReservedTime = 48
brokerRole = ASYNC_MASTER
flushDiskType = ASYNC_FLUSH
namesrvAddr=192.168.1.212:9876
autoCreateTopicEnable=true
listenPort = 10911
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH

```

这里需要根据自己的实际环境做适当修改
`注意：brokerIP1，namesrvAddr，这2个参数，一般配置为内网ip，提供内网访问。如果需要公网访问，这里一定要配置公网ip，否则无法访问。`





##### 编辑docker-compose.yml文件

```yaml
version: '3.5'
services:
  rmqnamesrv:
    image: apache/rocketmq:4.9.4
    container_name: rmqnamesrv
    ports:
      - 9876:9876
    volumes:
      - ./nameserver/logs:/home/rocketmq/logs
    command: sh mqnamesrv
    networks:
      rmq:
        aliases:
          - rmqnamesrv


  rmqbroker:
    image: apache/rocketmq:4.9.4
    container_name: rmqbroker
    ports:
      - 10911:10911
    volumes:
      - ./broker/logs:/home/rocketmq/logs
      - ./broker/store:/home/rocketmq/store
      - ./broker/broker.conf:/home/rocketmq/broker.conf
    environment:
      TZ: Asia/Shanghai
      NAMESRV_ADDR: "192.168.1.212:9876"
      JAVA_OPTS: " -Duser.home=/opt"
      JAVA_OPT_EXT: "-server -Xms256m -Xmx256m -Xmn256m"
    command: sh mqbroker -c /home/rocketmq/broker.conf
    links:
      - rmqnamesrv:rmqnamesrv
    networks:
      rmq:
        aliases:
          - rmqbroker
          
  rmqconsole:
    image: apacherocketmq/rocketmq-dashboard
    container_name: rmqconsole
    ports:
      - 8087:8080
    environment:
      JAVA_OPTS: -Drocketmq.namesrv.addr=rmqnamesrv:9876 -Dcom.rocketmq.sendMessageWithVIPChannel=false -Drocketmq.config.accessKey=rocketmq2 -Drocketmq.config.secretKey=12345678
    volumes:
      - ./console-ng/data:/tmp/rocketmq-console/data
    networks:
      rmq:
        aliases:
          - rmqconsole
networks:
  rmq:
    name: rmq
    driver: bridge



```

##### 执行docker-compose

```

docker-compose up -d

Starting rmqnamesrv   ... done
Starting rmqconsole   ... done
Starting rmqbroker    ... done

```



查看进程，使用命令docker-compose ps，确保State状态为up

```
docker-compose ps
```



访问控制台

http://192.168.1.212:8087/#/



