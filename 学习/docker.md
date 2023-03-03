
阿里云：5Beidangziqiang

- **docker镜像加速器配置**
	- https://cr.console.aliyun.com/cn-hangzhou/instances/mirrors

docker update redis --restart=always

## docker 启动mysql
```bash
docker pull mysql:5.7

docker run -p 13306:3306 --name mysql \
-v /mydata/mysql/log:/var/log/mysql \
-v /mydata/mysql/data:/var/lib/mysql \
-v /mydata/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7

参数说明
-p 13306:3306：将容器的 3306 端口映射到主机的 13306 端口
-v /mydata/mysql/conf:/etc/mysql：将配置文件夹挂载到主机
-v /mydata/mysql/log:/var/log/mysql：将日志文件夹挂载到主机
-v /mydata/mysql/data:/var/lib/mysql/：将配置文件夹挂载到主机
-e MYSQL_ROOT_PASSWORD=root：初始化 root 用户的密码
```
**mysql配置文件**
```bash
vi /mydata/mysql/conf/my.cnf

***********************
[client]
default_character_set=utf8
[mysql]
default-character-set=utf8
[mysqld]
collation_server = utf8_general_ci
character_set_server = utf8
skip-character-set-client-handshake
skip-name-resolve


```

## docker 启动redis
```bash
docker pull redis:6.0.8

mkdir -p /mydata/redis/conf
touch /mydata/redis/conf/redis.conf

docker run -p 6379:6379 --privileged=true --name redis -v /mydata/redis/data:/data \
-v /mydata/redis/conf/redis.conf:/etc/redis/redis.conf \
-d redis:6.0.8 redis-server /etc/redis/redis.conf

# 测试redis是否启动
docker exec -it redis redis-cli
```