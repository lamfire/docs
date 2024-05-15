# 搭建svn服务器

#### 1.下载镜像，创建并启动容器

```
mkdir -p /data/svn/svn_repo
cd /data/svn

docker pull elleflorio/svn-server

docker run --name svn-server -d --restart=always -v /data/svn/svn_repo:/home/svn -p 80:80 -p 3690:3690 elleflorio/svn-server

```

#### 2.配置SVN

进入浏览器管理页配置 http://192.168.1.214/svnadmin/  

按下面配置相关信息：

```
Subversion 授权文件              /etc/subversion/subversion-access-control                Test passed.

数据提供方相关
User view provider type:	passwd
User edit provider type:	passwd
Group view provider type:	svnauthfile
Group edit provider type:	svnauthfile
Repository view provider type:	svnclient
Repository edit provider type:	svnclient

用户身份验证
用户身份验证文件 (SVNUserFile)    /etc/subversion/passwd    Test passed.

Subversion 设置相关
代码仓库的父目录 (SVNParentPath)                /home/svn             Test passed.
'svn.exe' 或 'svn'可执行文件：                  /usr/bin/svn          Test passed.
'svnadmin.exe' 或 'svnadmin' 可执行文件：       /usr/bin/svnadmin     Test passed.
```

点击保存配置



#### 3.管理

后台创建仓库，用户等。一定要记得为用户分配RW权限，默认是没有W权限的。



#### 4.浏览仓库内容

地址：http://192.168.1.214/svn



#### 5.读取文件

的不同版本URL中加上参数p（如下为获取第三个版本）：

http://192.168.1.214/svn/test/linf_demo1.txt?p=3



### docker-compose

```
version: '3.6'
services:
  svn:        
    image: lamfire8390/svn-server:latest
    container_name: svn-server
    ports:  
        - "80:80"
        - "3690:3690"
    volumes:  
        - ./svn_repo:/home/svn 
        - /etc/localtime:/etc/localtime:ro 
    restart: always

```

配置

```
docker exec -it svn-server touch /etc/subversion/subversion-access-control       #授权文件
docker exec -it svn-server touch /etc/subversion/passwd                          #用户身份验证文件
docker exec -it svn-server chown -R apache:apache /etc/subversion                #设置权限
docker exec -t svn-server htpasswd -b /etc/subversion/passwd admin admin         #创建管理员
```

登录svnadmin设置即可使用。



#### 允许匿名读

```
docker exec -it svn-server sh

#修改内容
vi /etc/apache2/conf.d/dav_svn.conf
<Location /svn>
     DAV svn
     SVNParentPath /home/svn
     SVNListParentPath On
     AuthType Basic
     AuthName "Subversion Repository"
     AuthUserFile /etc/subversion/passwd
     AuthzSVNAccessFile /etc/subversion/subversion-access-control
     <LimitExcept GET PROPFIND OPTIONS REPORT>
       Require valid-user
     </LimitExcept>
</Location>/

#退出并重启
docker restart svn-server
```

