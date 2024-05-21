## frp配置内网穿透



### 服务端配置

**下载frp库并解压安装**

```
wget https://github.com/fatedier/frp/releases/download/v0.58.0/frp_0.58.0_linux_amd64.tar.gz
tar -zcvf frp_0.58.0_linux_amd64.tar.gz
cd frp_0.58.0_linux_amd64
cp frps /usr/local/bin

```

启动服务端

```
frps -p 7000 --dashboard-port 7777 --vhost-http-port 8888
```

说明：服务端口=7000，管理后台端口=7777，http代理端口=8888



**验证服务端是否启动成功**

```
打开网站，登录名和密码均为‘admin’,能进入则服务端启动成功
http://101.254.35.180:7777/
```



### 客户端配置

#### **下载frp库并解压安装**

```
wget https://github.com/fatedier/frp/releases/download/v0.58.0/frp_0.58.0_linux_amd64.tar.gz
tar -zcvf frp_0.58.0_linux_amd64.tar.gz
cd frp_0.58.0_linux_amd64
cp frpc /usr/local/bin
cp frpc.toml /etc/
```

#### 配置代理

```
vim /etc/frpc.toml

serverAddr = "101.254.35.180"
serverPort = 7000

[[proxies]]
name = "web1"
type = "http"
customDomains = ["disk.avator.cn"]
localIP = "192.168.1.10"
localPort = 80

[[proxies]]
name = "web2"
type = "http"
customDomains = ["plik.avator.cn"]
localIP = "192.168.1.80"
localPort = 8888
```

以上配置了两个http代理，分别访问本地网络中不同的WEB服务



#### 启动客户端

```
/usr/local/bin/frpc -c /etc/frpc.toml &
```



#### 配置域名解析hosts

```
101.254.35.180  disk.avator.cn
101.254.35.180  plik.avator.cn
```



#### 访问代理

```
http://disk.avator.cn:8888
```

