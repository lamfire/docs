# 谷歌地图代理nginx配置



谷歌地图访问地址

```
https://mt0.google.com/maps/vt?lyrs=s%40699&hl=zh-CN&gl=CN&&x=0&y=0&z=1
```



### 步骤1：安装Nginx

首先，确保系统上已经安装了Nginx。如果还没有安装，可以使用以下命令进行安装：

#### 在Ubuntu上安装Nginx：

```sh
sudo apt update
sudo apt install nginx
```

### 步骤2：配置Nginx代理

编辑Nginx的配置文件，通常是位于 `/etc/nginx/nginx.conf` 或者 `/etc/nginx/sites-available/default`。

在配置文件中添加一个服务器块来处理谷歌地图的代理请求：

```nginx
server {
    listen 80;
    server_name your_domain.com;  # 这里你可以使用你的域名或者服务器IP

    location /maps/ {
        proxy_pass https://mt0.google.com/maps/;
        proxy_set_header Host mt0.google.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```



### 步骤3：重启Nginx

保存配置文件并重启Nginx以应用更改：

```sh
sudo nginx -t  # 检查配置是否正确
sudo systemctl restart nginx  # 重启Nginx服务
```

### 步骤4：测试配置

打开你的浏览器，访问 `http://your_domain.com/maps/vt?lyrs=s%40699&hl=zh-CN&gl=CN&&x=0&y=0&z=1` (替换成适当的API Key)，检查是否可以正常加载谷歌地图。

### 额外补充：配置HTTPS (Optional)

为了安全起见，建议配置HTTPS，使用免费的Let’s Encrypt证书可以轻松实现。

#### 安装Certbot：

在Ubuntu上：

```sh
sudo apt install certbot python3-certbot-nginx
```



#### 获取证书并自动配置HTTPS：

```sh
sudo certbot --nginx -d your_domain.com
```

#### 更新Nginx配置以监听443端口并重定向HTTP到HTTPS：

```nginx
server {
    listen 80;
    server_name your_domain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name your_domain.com;

    ssl_certificate /etc/letsencrypt/live/your_domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your_domain.com/privkey.pem;

    location /maps/ {
        proxy_pass https://maps.googleapis.com/maps/;
        proxy_set_header Host maps.googleapis.com;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

最后，重新加载或重启Nginx服务：

```sh
sudo nginx -t
sudo systemctl restart nginx
```

通过上面的配置，你就能够使用Nginx作为代理来访问谷歌地图，并且可以通过HTTPS安全访问。