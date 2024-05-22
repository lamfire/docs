### nginx配置HTTPS 

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

通过上面的配置，你就能够使用HTTPS安全访问。