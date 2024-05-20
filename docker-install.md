# 安装docker



#### 1.卸载旧版

```
sudo apt remove docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc
```



#### 2.配置docker源

```
# Add Docker's official GPG key:
sudo apt-get install ca-certificates curl
sudo curl -fsSL https://mirrors.ustc.edu.cn/docker-ce/linux/ubuntu/gpg | sudo apt-key add -

# Add the repository to Apt sources:
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

sudo apt-get update
```

#### 3.安装

```
sudo apt-get install docker-ce docker-compose docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo systemctl daemon-reload 
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl status docker
```

#### 4.运行

```
docker pull lamfire8390/svn-server:latest
docker run --name svn-server -d --restart=always -v /etc/localtime:/etc/localtime -p 80:80 -p 3690:3690 lamfire8390/svn-server:latest
```

