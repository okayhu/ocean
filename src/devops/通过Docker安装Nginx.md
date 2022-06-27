## 环境

centos 7

## 步骤

```sh
docker run --name nginx -p 80:80 -d nginx:latest
```

如果 80 端口未开放，执行以下命令

```sh
firewall-cmd --premanent --add-port=80/tcp

# 也可以以service的方式开启
firewall-cmd --premanent --add-service=http

# 重新加载配置
firewall-cmd --reload

# 查看所有开启的端口
firewall-cmd --list-port

# 查看所有开启的服务
firewall-cmd --list-service
```

将 nginx 的配置文件以及 html 文件拷贝到宿主机

```sh
# 将 /usr/local/docker/nginx/ 换成你想存放的目录
mkdir -p /usr/local/docker/nginx/{conf,log,html}

docker cp nginx:/etc/nginx/nginx.conf /usr/local/docker/nginx/conf/nginx.conf

docker cp nginx:/etc/nginx/conf.d /usr/local/docker/nginx/conf/conf.d

docker cp nginx:/usr/share/nginx/html /usr/local/docker/nginx/html
```

停止并删除 nginx 容器

```sh
docker stop nginx

docker rm nginx

# 可以删除还在运行的容器
docker rm -f nginx
```

重新启动 nginx 容器

```sh
docker run --name nginx -p 80:80 \
-v /usr/local/docker/nginx/conf/nginx.conf:/etc/nginx/nginx.conf \
-v /usr/local/docker/nginx/conf/conf.d:/etc/nginx/conf.d \
-v /usr/local/docker/nginx/html:/usr/share/nginx/html \
-v /usr/local/docker/nginx/log:/var/log/nginx \
-d nginx:latest
```

 如果要修改 nginx 配置，可以直接修改 `usr/local/docker/nginx/conf/nginx.conf` 文件，并执行

```sh
docker restart nginx
```