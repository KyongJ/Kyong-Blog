---
title: Docker简易项目部署流程
author: Kyong
date: 2022-03-11
tag: 
  - 项目
---
# Docker简易项目部署流程

### 一、打包后端项目jar包

![image-20221230150232413](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20221230150232413.png)

- 点击右侧maven插件 -> package

- 打包成功后会在target目录下生成jar包

### 二、编写Dockerfile文件

```sh
FROM java:8
VOLUME /tmp
ADD blog-0.0.1-SNAPSHOT.jar blog.jar       
ENTRYPOINT ["java","-Djava.security.egd=file:/dev/./urandom","-jar","/blog.jar"] 
```

### 三、编写blog-start.sh脚本

```sh
#源jar路径  
SOURCE_PATH=/usr/local/docker/blog
#docker 镜像/容器名字或者jar名字 这里都命名为这个
SERVER_NAME=blog.jar
TAG=latest
SERVER_PORT=9090
#容器id
CID=$(docker ps | grep "$SERVER_NAME" | awk '{print $1}')
#镜像id
IID=$(docker images | grep "$SERVER_NAME:$TAG" | awk '{print $3}')
if [ -n "$CID" ]; then
  echo "存在容器$SERVER_NAME, CID-$CID"
  docker stop $SERVER_NAME
  docker rm $SERVER_NAME
fi
# 构建docker镜像
if [ -n "$IID" ]; then
  echo "存在$SERVER_NAME:$TAG镜像，IID=$IID"
  docker rmi $SERVER_NAME:$TAG
else
  echo "不存在$SERVER_NAME:$TAG镜像，开始构建镜像"
  cd $SOURCE_PATH
  docker build -t $SERVER_NAME:$TAG .
fi
# 运行docker容器
docker run --name $SERVER_NAME -d -p $SERVER_PORT:$SERVER_PORT $SERVER_NAME:$TAG
echo "$SERVER_NAME容器创建完成"
```

> **ps：sh文件需要用notepad++转为Unix格式**

### 四、将文件传输到服务器

![image-20221230150611240](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20221230150611240.png)

 将上述三个文件传输到/usr/local/docker下（手动创建文件夹）

### 五、docker运行后端项目

进入服务器/usr/local/docker下，构建后端镜像

```sh
sh ./blog-start.sh 
```

![image-20221230150715383](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20221230150715383.png)

**ps：需要重新部署只需重新传jar包，执行sh脚本即可**

### 六、打包前端项目

```sh
npm run build
```

打包成功后会在目录下生成dist文件

将React打包项目传输到/usr/local/react下

![image-20221230150901041](https://kyong-blog.oss-cn-shenzhen.aliyuncs.com/articleContent/image-20221230150901041.png)

### 七、Nginx配置

```sh
user  root;
worker_processes  1;

error_log  /var/log/nginx/error_nginx.log debug;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
    error_log   /var/log/nginx/error_nginx.log debug;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;
                include /etc/nginx/conf.d/*.conf;

    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_comp_level 4;
    gzip_types text/plain application/x-javascript text/css application/xml text/javascript application/x-httpd-php image/jpeg image/gif image/png;
    gzip_vary on;

     server{
        listen 80;
        server_name www.kyong.site;

        location /{
            root /usr/local/react/blog;
            index index.html index.htm;
            try_files $uri $uri/ /index.html;
        }

        location ^~ /api/ {
            proxy_pass http://39.108.56.44:9090/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }

     server {
        listen  80;
        server_name  www.admin.kyong.site;

        location / {
            root   /usr/local/react/admin;
            index  index.html index.htm;
            try_files $uri $uri/ /index.html;
        }

        location ^~ /api/ {
            proxy_pass http://39.108.56.44:9090/;
            proxy_set_header   Host             $host;
            proxy_set_header   X-Real-IP        $remote_addr;
            proxy_set_header   X-Forwarded-For  $proxy_add_x_forwarded_for;
        }


    }

}
```

> docker 启动nginx

```sh
docker run --name blog_nginx --restart=always -p 80:80 -p 443:443 -d -v /usr/local/nginx/nginx.conf:/etc/nginx/nginx.conf -v /usr/local/react:/usr/local/react -v  nginx 
```
















