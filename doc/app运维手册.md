# 服务器

|     ip      | 用户名 |    密码     | 硬盘 | 内存 | 用途                                                         |
| :---------: | :----: | :---------: | :--: | :--: | ------------------------------------------------------------ |
| 10.67.32.22 |  root  | cpos/123456 | 500  |  16  |                                                              |
| 10.67.32.12 |  root  |             |      |      | Avx2 机器学习训练                                            |
| 10.67.31.22 |  root  |             |      |      | 大连环境中间件:Apisix: http://10.67.31.22:9000/  admin/admin |
| 10.67.32.11 |  root  |             |      |      | 应用服务器                                                   |





# 中间件服务端口

|  中间件名称   |     ip      |    端口     |                备注                 |      |
| :-----------: | :---------: | :---------: | :---------------------------------: | ---- |
|   sqlserver   | 10.67.31.22 |    1433     |   SA /Q1w2e3r4 mnctuser/Admin1234   |      |
|   zookeeper   | 10.67.31.22 |    2181     |                                     |      |
|     kafka     | 10.67.31.22 |    9092     |                                     |      |
|     codis     | 10.67.31.22 |    29000    | ./redis-cli -h 10.67.31.22 -p 29000 |      |
|     mysql     | 10.67.31.22 |    3306     |             root/123456             |      |
|    apollo     | 10.67.31.22 |    8070     |            apollo/admin             |      |
|    jenkins    | 10.67.31.22 |    7766     |            admin/123456             |      |
|    pcp web    | 10.67.31.22 |    8818     |                                     |      |
|     redis     | 10.67.31.22 |    6379     |             密码:123456             |      |
|   rabbitmq    | 10.67.31.22 | 5672/15672  |            admin/123456             |      |
|    pulsar     | 10.67.31.22 | 26650/28808 |                                     |      |
|    xxl-job    | 10.67.31.22 |    18080    |            admin/123456             |      |
|     doris     | 10.67.32.12 |    9030     |                                     |      |
|     nginx     | 10.67.31.22 |     80      |        /usr/local/nginx/conf        |      |
|    mongodb    | 10.67.31.22 |    27017    |                                     |      |
|    Eureka     | 10.67.31.22 |    8761     |                                     |      |
| redis集群模式 | 10.67.31.22 | 50001-50006 |         密码：Hitachi@2021          |      |
| 目标检测服务  | 10.67.31.22 |    9988     |                                     |      |
|   智能餐厅    | 10.67.31.22 |    8081     |                                     |      |
|  mc管理平台   | 10.67.31.22 |    8828     |             100051547/1             |      |
|  wildfly管理  | 10.67.31.22 |    19990    |           admin/12345678            |      |

# sqlserver安装

### docker安装

```
docker pull mcr.microsoft.com/mssql/server:2017-latest
```

### 运行容器

```
docker run -e "ACCEPT_EULA=Y" -e "SA_PASSWORD=Q1w2e3r4" -p 1433:1433 --name sqlserver2017 -d mcr.microsoft.com/mssql/server:2017-latest
```

### 进入容器

##### 进入容器

```
docker exec -it sqlserver2017 /bin/bash
```

##### 连接数据库

```
/opt/mssql-tools/bin/sqlcmd -S localhost -U SA -P "Q1w2e3r4"
```

查看数据库

```
select name from sys.Databases go 
go
```

# Kafka安装

### 打开防火墙端口

```
firewall-cmd --add-port=2181/tcp --permanent
firewall-cmd --add-port=9092/tcp --permanent
firewall-cmd --reload

systemctl restart docker
firewall-cmd --query-port=2181/tcp
firewall-cmd --query-port=9092/tcp
```



### zookeeper安装

```
docker run -dit --restart=always --log-driver json-file --log-opt max-size=100m --log-opt max-file=2  --name zookeeper \
-p 2181:2181 \
-v /etc/localtime:/etc/localtime \
-t wurstmeister/zookeeper
```



### kafka安装

```

docker run -dit --restart=always --log-driver json-file --log-opt max-size=100m --log-opt max-file=2 --name kafka \
-p 9092:9092 \
-e KAFKA_BROKER_ID=0 \
-e KAFKA_ZOOKEEPER_CONNECT=10.67.31.22:2181 \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://10.67.31.22:9092 \
-e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092 wurstmeister/kafka
```



### kafka 验证

```
docker exec -it kafka /bin/sh
cd /opt/kafka_2.11-2.0.0/bin
./kafka-console-producer.sh --broker-list localhost:9092 --topic sun
{“datas”:[{“channel”:"",“metric”:“temperature”,“producer”:“ijinus”,“sn”:“IJA0101-00002245”,“time”:“1543207156000”,“value”:“80”}],“ver”:“1.0”}
```

# codis安装

### 下载源码

```
git clone https://github.com/CodisLabs/codis.git -b release3.2
```

### 修改Dockerfile

```
FROM golang:1.10.3-alpine3.8 as builder

ENV GOPATH /go
ENV CODIS  ${GOPATH}/src/github.com/CodisLabs/codis
ENV PATH   ${GOPATH}/bin:${PATH}:${CODIS}/bin
COPY . ${CODIS}

RUN echo -e "https://mirrors.aliyun.com/alpine/v3.8/main/\nhttps://mirrors.aliyun.com/alpine/v3.8/community/" > /etc/apk/repositories ;\
    apk add --no-cache --virtual .build-deps \
        make \
        bash \
        gcc \
        musl-dev \
        autoconf \
        linux-headers \
    ; \
    make -C ${CODIS} distclean ;\
    make -C ${CODIS} build-all ;\
    apk del .build-deps

FROM alpine:3.8
ENV PATH   ${PATH}:/codis/bin

RUN echo -e "https://mirrors.aliyun.com/alpine/v3.8/main/\nhttps://mirrors.aliyun.com/alpine/v3.8/community/" > /etc/apk/repositories ;\
    apk add --no-cache tzdata ;\
    cp /usr/share/zoneinfo/Asia/Shanghai /etc/localtime

COPY --from=builder /go/src/github.com/CodisLabs/codis /codis

WORKDIR /codis

```

### 生成镜像

```
docker build -f Dockerfile -t codis-image .
```

### 修改配置文件 config/dashboard.toml

```
#coordinator_name = "filesystem"
#coordinator_addr = "/tmp/codis"
coordinator_name = "zookeeper"
coordinator_addr = "10.67.31.22:2181"
#coordinator_auth = ""
```

### 修改配置文件 config/proxy.toml

```
jodis_name = "zookeeper"
jodis_addr = "10.67.31.22:2181"
jodis_auth = ""
jodis_timeout = "20s"
jodis_compatible = false
```

### 修改配置文件 config/redis.conf

```
protected-mode no
dir "/codis"
docker启动codis-server就会失败，一定要去除参数 daemonize yes 
```

### 修改docker.sh

```
server)
    for ((i=0;i<4;i++)); do
        let port="26379 + i"
        docker rm -f      "Codis-S${port}" &> /dev/null
        docker run --net=host --name "Codis-S${port}" -d \
            -v `realpath ../config/redis.conf`:/codis/redis.conf \
            -v `realpath log`:/codis/log \
            codis-image \
            codis-server redis.conf  --logfile log/${port}.log --port ${port}
    done
    ;;
```

### 启动应用

```
sh docker.sh zookeeper  如果已经有了，可以不启动
sh docker.sh dashboard
sh docker.sh fe
sh docker.sh proxy
sh docker.sh server
```

### 启动3个sentinel

```
docker run --net=host --name "Codis-T46380" -d \
            -v `realpath ../config/sentinel.conf`:/codis/sentinel.conf \
            -v `realpath log`:/codis/log \
            codis-image \
            redis-sentinel sentinel.conf  --port 46380 --protected-mode no

docker run --net=host --name "Codis-T46381" -d \
            -v `realpath ../config/sentinel.conf`:/codis/sentinel.conf \
            -v `realpath log`:/codis/log \
            codis-image \
            redis-sentinel sentinel.conf  --port 46381 --protected-mode no
docker run --net=host --name "Codis-T46382" -d \
            -v `realpath ../config/sentinel.conf`:/codis/sentinel.conf \
            -v `realpath log`:/codis/log \
            codis-image \
            redis-sentinel sentinel.conf  --port 46382 --protected-mode no

```

```
https://www.cnblogs.com/yeyongjian/p/13328220.html
```

### codis的前端配置页面

```
http://10.67.31.22:38080/#codis-demo
```

# Jenkins 部署

```
http://10.67.31.22:7766/
```

# apollo部署

##### 启动目录

```
sh /opt/software/apollo-build-scripts/demo.sh start
```

##### 访问地址

```
http://10.67.31.22:8070
username:apollo
passwd: admin
```

# mysql5.7 部署

```
docker run -p 3306:3306 --name mysql -v /opt/software/mysql/conf:/etc/mysql/conf.d -v /opt/software/mysql/logs:/logs -v /opt/software/mysql/data:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=123456 -d mysql:5.7
```


# Redis 部署

### redis 启动

```
docker run --restart=always --log-opt max-size=100m --log-opt max-file=2 -p 6379:6379 --name myredis -v /opt/software/redis/myredis.conf:/etc/redis/redis.conf -v /opt/software/redis/data:/data -d redis redis-server /etc/redis/redis.conf  --appendonly yes
```

### 密码

```
123456
```

# pulsar 部署

### docker启动：

```
docker run -itd \
    --name pulsar-standalone \
    -p 26650:6650 \
    -p 28088:8080 \
    apachepulsar/pulsar:2.7.2 \
    sh -c  "bin/pulsar standalone > pulsar.log 2>&1 & \
            sleep 30 && bin/pulsar-admin clusters update standalone \
            --url http://pulsar-standalone:8080 \
            --broker-url pulsar://pulsar-standalone:6650 & \
            tail -F pulsar.log"
```

### `pulsar-manager`容器

```
docker run -itd \
    --name pulsar-manager \
    -p 9527:9527 -p 7750:7750 \
    -e SPRING\_CONFIGURATION\_FILE=/pulsar-manager/pulsar-manager/application.properties \
    --link pulsar-standalone \
    --entrypoint="" \
    apachepulsar/pulsar-manager:v0.2.0 \
    sh -c  "sed -i '/^default.environment.name/ s|.\*|default.environment.name=pulsar-standalone|' /pulsar-manager/pulsar-manager/application.properties & \
            sed -i '/^default.environment.service\_url/ s|.\*|default.environment.service\_url=http://pulsar-standalone:8080|' /pulsar-manager/pulsar-manager/application.properties & \
            /pulsar-manager/entrypoint.sh & \
            tail -F /pulsar-manager/pulsar-manager/pulsar-manager.log"
```

### 应用访问端口：

```
pulsar://10.67.31.22:26650
```

### 管理端口

```
http://pulsar-standalone:8080
```

# Rabbitmq 部署

```
docker run -d --hostname my-rabbit --name rabbit -p 15672:15672 -p 5672:5672 rabbitmq
```

### 插件启用

```
docker exec -it 容器id /bin/bash
rabbitmq-plugins enable rabbitmq_management
```

### 创建用户名密码

```
rabbitmqctl add_user admin 123456
rabbitmqctl set_user_tags admin administrator
```



### 后台登录

```
http://10.67.31.22:15672/
```

#### 用户名密码

```
admin/123456
```



# XXL-JOB 部署

```
docker run -e PARAMS="--spring.datasource.url=jdbc:mysql://10.67.31.22:3306/xxl_job?Unicode=true&characterEncoding=UTF-8&useSSL=false --spring.datasource.username=root --spring.datasource.password=123456 --xxl.admin.login=false" -p 18080:8080 --name xxl-job-admin -d xuxueli/xxl-job-admin:2.2.0
```

### 访问地址

```
   http://10.67.31.22:18080/xxl-job-admin
```

### 用户名密码

```
admin/123456
```

# Mongodb 部署

```
docker run -d -p 27017:27017 --name example-mongo mongo:latest
```

# Redis Cluster

### 手动启动

```
cd /opt/software/redis-5.0.8/utils/create-cluster
./create-cluster start
```



# Eureka 部署

```
 docker run -dit --name eureka -p 8761:8761  springcloud/eureka:latest
```
### 访问地址
```
http://10.67.31.22:8761/
```
------

------

# nginx 编译rtmp/flv 模块

### 编译配置

```
./configure --add-module=../nginx-rtmp-module --add-module=../nginx-http-flv-module --prefix=/usr/local/nginx --with-http_ssl_module --with-http_v2_module --with-http_dav_module --with-http_stub_status_module --with-threads --with-file-aio --with-http_v2_module --with-http_realip_module --with-stream_ssl_preread_module --with-http_addition_module --with-http_sub_module --with-http_dav_module --with-http_flv_module --with-http_mp4_module --with-http_gunzip_module --with-http_gzip_static_module --with-http_random_index_module --with-http_secure_link_module --with-http_degradation_module --with-http_slice_module --with-http_stub_status_module --with-http_auth_request_module --with-stream
```

### rtmp nginx 推拉流配置

```
rtmp {
    server {
        listen 1935;
        chunk_size 4096;
        application vod {
            play /opt/softwarevideo/rtmp/vod;
        }
        application live {
            live on;#直播模式        
	}
        application hls {
            live on;#直播模式
            hls on; #这个参数把直播服务器改造成实时回放服务器。
            wait_key on; #对视频切片进行保护，这样就不会产生马赛克了。
            hls_path /opt/software/video/rtmp/hls/; #切片视频文件存放位置。
            hls_fragment 10s;     #每个视频切片的时长。
            hls_playlist_length 60s;  #总共可以回看的事件，这里设置的是1分钟。
            hls_continuous on; #连续模式。
            hls_cleanup on;    #对多余的切片进行删除。
            hls_nested on;     #嵌套模式。
            # disable consuming the stream from nginx as rtmp
            deny play all;
         }
     }
}
```

```
   server {   
        listen       80;
        server_name  localhost;
        location /stat {
            rtmp_stat all;
            rtmp_stat_stylesheet stat.xsl;
        }
        location /stat.xsl {
            root /data/python/software/nginx-rtmp/nginx-rtmp-module-1.2.1;
        }
        location /hls {
            # Disable cache
            add_header Cache-Control no-cache;
            # CORS setup
            add_header 'Access-Control-Allow-Origin' '*' always;
            add_header 'Access-Control-Expose-Headers' 'Content-Length';

            # allow CORS preflight requests
            if ($request_method = 'OPTIONS') {
                add_header 'Access-Control-Allow-Origin' '*';
                add_header 'Access-Control-Max-Age' 1728000;
                add_header 'Content-Type' 'text/plain charset=UTF-8';
                add_header 'Content-Length' 0;
                return 204;
            }
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /home/video/rtmp;
        }
        location /vod {
            root   /home/video/rtmp;
        }
        location / {
            root   html;
            index  index.html index.htm;
        }
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    } 

```

### nginx隧道建立

服务端

```
stream {
    upstream tunnel
    {
        hash $remote_addr consistent;
        server localhost:8888 weight=5 max_fails=3 fail_timeout=30s;
    }
    server
    {
        listen 8888;
        proxy_connect_timeout 30s;
        proxy_timeout 30s;
        proxy_pass tunnel;
    }

}
```

客户端建立隧道

```
ssh -C -f -N -g -R 8888:localhost:8888 -l root 10.67.32.12
```

服务端连接隧道代理

```
export http_proxy=http://127.0.0.1:8888
export https_proxy=http://127.0.0.1:8888
```



### docker 批量增加自启动

```
docker update   --restart=always $(docker ps -aq)
```



# --------------------------应用服务----------------------------

# PCP 应用

### 应用路径

```
/opt/app/pcp/
```

```
http://10.67.31.22:8188/
```

用户名密码

```
admin/123456
```



# menu-plus

### 应用路径

```
/opt/app/mc
```

### 服务端口

```
menu-basic   8883
menu-job     8881
menu-monitor 8886
menu-regular 8880
menu-sellout 8885
menu-store   8882
menu-output  8884
```

# 餐厅智能

### 图片服务

### 代码修改

```
/smartRestaurant/smart/model/detect_image_op.py
data1["photo_url"] = "http://10.67.31.22:9987/" + image_name
data1["video_url"] = "http://10.67.31.22:9987/1.mp4"
```

### 修改数据库连接

```
/smartRestaurant/smartRestaurant/smart/model/db_common/database.py
```



### nginx添加配置

```
server {
    listen       9987;
    server_name  127.0.0.1;
    access_log  /var/log/nginx/file-server.access.log;
    location / {
        root   /opt/app/smartRestaurant/smartRestaurant/smart/data/output;
        autoindex on;
        autoindex_exact_size on;
        autoindex_localtime on;
        index  index.html index.htm;
    }
 }   
```



### rtmp视频推拉流

#### ip摄像头连接本地网路由，推视频流

```
ip摄像头地址：10.42.0.88 admin/123456
```

由于服务器无法拉取本地视频数据，需要通过本地网推送到服务器

```
主码流：rtmp://10.42.0.1:19361/live/smart01
子码流配置：rtmp://10.42.0.1:1936/live/smart01
```

#### 本地视频推送

```
客户端中转ip： 10.42.0.1
```

#### nginx配置视频流推送

```
     upstream rtsp_push_server
    {
        hash $remote_addr consistent;
        server 10.67.31.22:1935 weight=5 max_fails=3 fail_timeout=30s;
    }
    server
    {
        listen 1936;
        proxy_connect_timeout 30s;
        proxy_timeout 30s;
        proxy_pass rtsp_push_server;
    }

```

#### 服务器RTMP拉流nginx配置

```
rtmp {
    server {
        listen 1935;
        chunk_size 4096;
        application vod {
            play /opt/softwarevideo/rtmp/vod;
        }
        application live {
            live on;#直播模式        
	}
        application hls {
            live on;#直播模式
            hls on; #这个参数把直播服务器改造成实时回放服务器。
            wait_key on; #对视频切片进行保护，这样就不会产生马赛克了。
            hls_path /opt/software/video/rtmp/hls/; #切片视频文件存放位置。
            hls_fragment 10s;     #每个视频切片的时长。
            hls_playlist_length 60s;  #总共可以回看的事件，这里设置的是1分钟。
            hls_continuous on; #连续模式。
            hls_cleanup on;    #对多余的切片进行删除。
            hls_nested on;     #嵌套模式。
            # disable consuming the stream from nginx as rtmp
            deny play all;
         }
     }
}
```



### 目标检测模型应用服务启动

```
nohup python ./manage.py runserver 0.0.0.0:9988 &
```

### vue 前端

#### web编译

```
npm run build:prod
```

#### nginx配置

```
server {
    listen       8081;
    server_name  10.67.31.22;

    #charset koi8-r;
    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /opt/app/smartRestaurant/smartRestaurantVue/dist;
        try_files $uri $uri/ /index.html;
        index  index.html index.htm;
    }

    location /prod-api/ {
	proxy_set_header Host $http_host;
	proxy_set_header X-Real-IP $remote_addr;
	proxy_set_header REMOTE-HOST $remote_addr;
	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
	proxy_pass http://localhost:9988/;
   }
}
```

### vue 访问地址

```
http://10.67.31.22:8081/
```

# AIOps 应用

### bentoml机器学习模型预测服务启动

#### api服务生成

```
/opt/app/AIOpsPOC/aiops/bentoml
python model_package_save.py
```

#### 启动api服务

```
bentoml list
nohup bentoml serve-gunicorn  BentoMlApiService:20220811135849_08A172 --port 9984 &
```

#### 测试

预测正常值

```
curl -i \
--header "Content-Type: application/json" \
--request POST \
--data '[{"viewId":"2021",
"viewName":"转化率",
"attrId":"2",
"attrName":"转化率",
"taskId":"1621851803322",
"window":180,
"time":"2021-05-17 17:28:00",
"dataC":"0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9",
"dataB":"0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9",
"dataA":"0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9,0.9"
}]' \
10.67.31.22:9984/predict
```

异常数据

```
curl -i \
--header "Content-Type: application/json" \
--request POST \
--data '[{"viewId":"2021",
"viewName":"转化率",
"attrId":"2",
"attrName":"转化率",
"taskId":"1621851803322",
"window":180,
"time":"2021-05-17 17:28:00",
"dataC":"3,3,3,0,1,1,2,0,0,3,1,1,0,1,2,2,5,0,2,0,2,0,0,0,2,4,3,3,2,1,7,6,7,15,7,4,4,2,4,1,5,6,3,2,4,3,2,2,3,1,5,2,0,1,0,1,1,0,0,1,0,0,1,1,1,0,1,1,0,4,1,0,1,0,1,0,1,0,0,0,1,2,2,2,4,1,3,2,2,0,2,0,0,1,0,1,2,2,2,1,1,1,1,0,1,1,0,3,1,0,3,1,1,0,0,0,2,1,0,1,2,0,0,1,2,1,1,1,1,0,0,0,1,1,2,0,0,1,1,2,2,3,2,3,2,1,1,2,0,1,2,4,0,1,0,2,0,0,0,1,1,1,2,1,1,0,2,3,0,0,0,1,1,0,1,2,1,2,0,0,1,0,3,1,0,1,2,1,0,1,2,0,0,0,0,0,1,0,1,1,3,4,2,2,1,1,1,2,1,1,0,2,2,2,2,1,0,0,3,0,2,15,2,1,0,2,0,1,3,3,0,1,0,2,1,2,1,0,3,2,1,2,0,1,0,3,2,1,2,1,5,2,4,1,0,3,0,0,0,0,2,1,3,2,0,2,2,5,1,3,0,1,1,1,1,1,1,1,6,3,1,1,2,8,4,0,1,3,2,1,2,1,0,5,3,2,3,5,32,39,19,9,1,1,1,1,18,54,13,9,6,3,4,4,3,2,0,2,1,1,5,4,3,3,1,3,2,3,2,0,4,1,1,0,1,1,1,3,3,1,1,4,1,4,1,4,2,1,1,3,2,1,1,1,1,2,3,1,4,3,1",
"dataB":"2,2,2,3,1,6,5,1,1,4,4,0,1,5,6,2,0,6,4,1,1,0,1,2,4,4,5,3,2,1,1,3,2,3,2,2,1,4,0,3,1,3,1,5,4,2,2,2,1,3,2,7,4,3,1,0,1,4,3,6,2,5,5,5,9,20,15,7,8,0,1,0,0,6,2,2,2,1,3,2,3,3,2,8,7,13,4,4,1,2,2,1,4,1,2,3,0,0,2,2,3,1,1,2,2,2,1,2,7,4,8,5,2,8,5,8,5,6,3,4,7,1,3,1,1,6,1,0,9,3,1,2,3,7,1,0,1,0,3,2,2,3,4,4,2,1,6,2,2,1,4,2,2,5,1,0,0,1,3,5,1,0,2,2,0,3,1,3,1,1,3,0,0,1,0,2,2,1,2,3,1,1,1,1,4,3,0,2,3,5,4,4,0,1,1,1,3,2,2,3,1,1,2,0,4,0,0,0,0,1,0,1,2,4,3,3,4,1,1,1,4,1,1,1,1,1,1,2,3,2,3,0,6,4,2,2,0,3,2,1,8,2,6,4,0,1,3,1,7,1,1,5,1,3,0,2,3,3,1,2,6,8,5,0,1,1,2,4,2,1,0,0,0,3,1,3,1,5,2,0,2,2,1,1,2,3,3,0,1,0,0,2,3,3,8,2,4,2,1,5,4,3,2,2,3,1,3,1,4,2,3,2,4,1,3,1,1,0,2,3,1,4,4,4,2,2,2,1,3,1,2,2,3,3,3,1,3,2,1,1,4,1,1,4,2,0,2,1,2,0,3,2,2,1,3,1,1,2,5,4,0",
"dataA":"7,14,9,6,11,15,5,13,19,18,8,6,7,10,11,7,18,15,11,13,11,18,11,9,21,9,9,11,7,8,10,9,11,8,10,13,9,8,10,10,9,11,8,8,7,4,3,4,4,9,9,3,4,9,5,2,6,4,5,3,5,3,11,10,5,3,5,7,4,9,7,3,13,4,15,6,2,5,4,4,1,3,7,3,1,2,20,7,5,9,3,5,1,2,4,2,8,3,1,5,5,0,8,6,6,4,1,3,5,2,3,1,2,4,2,3,1,2,3,2,0,3,3,0,1,0,3,3,1,4,6,2,2,0,3,5,2,5,9,3,0,3,1,0,2,3,3,9,20,6,5,8,22,88,35,16,10,5,6,72,58,12,31,38,30,23,19,8,10,18,4,8,14,12,157,286,325,174,193,453,498"
}]' \
10.67.31.22:9984/predict
```

#### 模型训练服务启动

```
nohup python ./manage.py runserver 0.0.0.0:9955 &
```

#### 模型训练

```
curl -H "Content-type: application/json" -X POST -d '{"trainOrTest": ["train","test"], "positiveOrNegative": "", "source": ["conv_rate"], "type": "2", "beginTime": 1621002720, "endTime": 1621399120 }' "http://10.67.31.22:9955/Train"
```

### 实时检测线上转化率数据

#### 应用启动

```
nohup java -jar /opt/app/AI-conv-rate/AI-conv-rate-server-1.0.jar &
```

#### kafka接入topic

```
conv-rate-topic
```

数据结构

```
public class ConvRateEntity {
    private int type;
    private int entry;
    private int pay;
    private  double rate;
    private  Date ts;
}
```

# MenuCenter 服务

### wildfly部署

#### 修改端口

/opt/app/wildfly-20.0.0.Final/standalone/configuration/standalone.xml

```
<socket-binding name="ajp" port="${jboss.ajp.port:18009}"/>
<socket-binding name="http" port="${jboss.http.port:8828}"/>
<socket-binding name="https" port="${jboss.https.port:18443}"/>
<socket-binding name="management-http" interface="management" port="${jboss.management.http.port:19990}"/>
<socket-binding name="management-https" interface="management" port="${jboss.management.https.port:19993}"/>
```

### 代码编译

```
mvn clean package -P local -Dmaven.test.skip=true
```

### 启动应用

```
nohup ./standalone.sh  >startWildFly.log 2>&1 &
```

### wildfly 管理地址

```
http://10.67.31.22:19990/console/index.html
```

### 服务地址

```
http://10.67.31.22:8828/menuCenter-local/
```

# 大数据服务部署端口

10.67.32.11 test/123456

|     服务      |     IP      | 端口  |       描述        |
| :-----------: | :---------: | :---: | :---------------: |
|   hdfs服务    | 10.67.32.11 | 9000  |                   |
| hdfs管理后台  | 10.67.32.11 | 50070 |                   |
| yarn管理后台  | 10.67.32.11 | 8088  |                   |
| flink管理后台 | 10.67.32.11 | 8082  |                   |
| Spark web管理 | 10.67.32.11 | 8085  |                   |
|   Hbase管理   | 10.67.32.11 | 16010 |                   |
|     mysql     | 10.67.32.11 | 3306  | root/Hitachi@2020 |
|     kafka     | 10.67.32.11 | 9092  |                   |
|               |             |       |                   |
|               |             |       |                   |
|               |             |       |                   |
|               |             |       |                   |
|               |             |       |                   |



# 大数据Hadoop2.7.7 部署

### 主机环境变量配置

```
export HADOOP_HOME=/home/bigdata/hadoop-2.7.7
export PATH=$HADOOP_HOME/bin:$PATH
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export  HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib:$HADOOP_COMMON_LIB_NATIVE_DIR"
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
export HADOOP_HOME_WARN_SUPPRESS=1
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export HIVE_HOME=/home/bigdata/apache-hive-2.3.4-bin
export PATH=$HIVE_HOME/bin:$PATH
#Sprak ENV
export SPARK_HOME=/home/bigdata/spark-2.2.3-bin-hadoop2.7
export PATH=$PATH:$SPARK_HOME/bin
export LIVY_HOME=/home/bigdata/apache-livy-0.7.0-incubating-bin
export CLASSPATH=$($HADOOP_HOME/bin/hadoop classpath):$CLASSPATH
```

## hadoop配置

```
配置路径：/home/bigdata/hadoop-2.7.7/etc/hadoop/
```

### hadoop-env.sh

```
export JAVA_HOME=/opt/java/zulu8.31.0.1-jdk8.0.181-linux_x64
```

### core-site.xml

```
<configuration>
  <property>
       <name>hadoop.tmp.dir</name>
       <value>file:///home/bigdata/hadoop-2.7.7/tmp</value>
       <description>Abase for other temporary directories.</description>
   </property>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://node01:9000</value>
   </property>
    <property>
        <name>hadoop.proxyuser.root.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.root.groups</name>
        <value>*</value>
    </property>
</configuration>
```

### hdfs-site.xml

```
<configuration>
   <!-- 以下是数据目录，根据分区情况自行调整-->
    <property>
        <name>dfs.namenode.name.dir</name>
        <value>/home/bigdata/hadoop-2.7.7/data/dfs/name</value>
    </property>
    <property>
        <name>dfs.datanode.data.dir</name>
        <value>/home/bigdata/hadoop-2.7.7/data/dfs/data</value>
    </property>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
    <property>
        <name>dfs.permissions.enabled</name>
        <value>false</value>
    </property>
</configuration>
```

### mapred-site.xml

```
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <!-- 这里指定了每个MR任务占用的最小和最大内存限制，按实际服务器配置调整，建议-Xmx为1/8的服务器内存 -->
    <property>
        <name>mapred.child.java.opts</name>
        <value>-Xms64m -Xmx1024m -Xloggc:/tmp/@taskid@.gc</value>
    </property>
    <!-- 这里指定了每个Mapper最大内存限制，建议为1/8的服务器内存 -->
    <property>
        <name>mapreduce.map.memory.mb</name>
        <value>1024</value>
    </property>
    <!-- 这里指定了每个Reduce最大内存限制，建议为1/8的服务器内存 -->
    <property>
        <name>mapreduce.reduce.memory.mb</name>
        <value>1024</value>
    </property>
    <property>
        <name>yarn.scheduler.minimum-allocation-mb</name>
        <value>64</value>
    </property>
</configuration>
```

### yarn-site.xml

```
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.vmem-pmem-ratio</name>
        <value>5.0</value>
    </property>
    <property>
       <name>yarn.nodemanager.vmem-check-enabled</name>
       <value>false</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>      
	<value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
```

### 格式化HDFS

```
hadoop namenode -format
```

### 启动Hadoop

```
/home/bigdata/hadoop-2.7.7/sbin/start-all.sh
```

## 访问地址

### hdfs地址

```
http://10.67.32.11:50070/
```

### yarn访问地址

```
http://10.67.32.11:8088/cluster
```

# Flink1.7.2 部署

### 配置文件修改

```
jobmanager.rpc.address: node01
jobmanager.rpc.port: 6123
jobmanager.heap.size: 1024m
taskmanager.heap.size: 1024m
taskmanager.numberOfTaskSlots: 1
parallelism.default: 1
jobmanager.web.port: 5566
##配置hadoop的配置文件
fs.hdfs.hadoopconf: /home/bigdata/hadoop-2.7.7/etc/hadoop
##访问hdfs系统使用的
fs.hdfs.hdfssite: /home/bigdata/hadoop-2.7.7/etc/hadoop/hdfs-site.xml
##配置checkpoint目录
state.backend.fs.checkpointdir: hdfs://node01:9000/flink-checkpoints
env.java.opts: -XX:+UseConcMarkSweepGC -XX:+PrintGCDetails -XX:-UseGCOverheadLimit -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/home/bigdata/wormhole-0.6.3/gc
yarn.application-attempts: 2
```

### 服务启动

```
/home/bigdata/flink-1.7.2/bin/start-cluster.sh
```

#### flink管理平台

```
http://10.67.32.11:8082
```

# Hive 部署

### 配置修改

### /home/bigdata/apache-hive-2.3.4-bin/conf/hive-env.sh

```
HADOOP_HOME=/home/bigdata/hadoop-2.7.7
export HIVE_CONF_DIR=/home/bigdata/apache-hive-2.3.4-bin/conf
export JAVA_HOME=/opt/java/zulu8.31.0.1-jdk8.0.181-linux_x64
export HIVE_HOME=/home/bigdata/apache-hive-2.3.4-bin
```

### hive-site.xml

```
  <property>
    <name>hive.repl.rootdir</name>
    <value>/home/bigdata/apache-hive-2.3.4-bin/repl/</value>
  </property>
  
  <property>
    <name>hive.exec.scratchdir</name>
    <value>/home/bigdata/apache-hive-2.3.4-bin/tmp/hive</value>
  </property>
  <property>
    <name>hive.repl.rootdir</name>
    <value>/home/bigdata/apache-hive-2.3.4-bin/repl/</value>
  </property>
  <property>
    <name>hive.exec.local.scratchdir</name>
    <value>/home/bigdata/apache-hive-2.3.4-bin/iotmp</value>
  </property>
  <property>
    <name>hive.downloaded.resources.dir</name>
    <value>/home/bigdata/apache-hive-2.3.4-bin/iotmp</value>
  </property>
  <property>
    <name>hive.scratch.dir.permission</name>
    <value>777</value>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>Hitachi@2020</value>
  </property>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://127.0.0.1:3306/hive234_new?createDatabaseIfNotExist=true</value>
  </property>
```

### /etc/profile 配置

```
export HIVE_HOME=/home/bigdata/apache-hive-2.3.4-bin
export PATH=$HIVE_HOME/bin:$PATH
```

### 初始化元数据库

```
/home/bigdata/apache-hive-2.3.4-bin/bin/schematool -dbType mysql -initSchema
```

### 启动hive

```
nohup hive --service hiveserver2 >> /home/bigdata/apache-hive-2.3.4-bin/logs/hiveserver2.out 2>&1 &

nohup hive --service metastore > /home/bigdata/apache-hive-2.3.4-bin/logs/hivemetastore.log  2>&1 &
```

# Spark部署

### spark-env.sh

```
export SPARK_DIST_CLASSPAT=$(/home/bigdata/hadoop-2.7.7/bin/hadoop classpath)
export JAVA_HOME=/opt/java/zulu8.31.0.1-jdk8.0.181-linux_x64/
export SCALA_HOME=/home/bigdata/scala-2.12.7/ #配置scala路径
export HADOOP_HOME=/home/bigdata/hadoop-2.7.7
export HADOOP_CONF_DIR=/home/bigdata/hadoop-2.7.7/etc/hadoop
export HIVE_HOME=/home/bigdata/apache-hive-2.3.4-bin
export HIVE_CONF_DIR=/home/bigdata/apache-hive-2.3.4-bin/conf
#spark
export SPARK_HOME=/home/bigdata/spark-2.2.3-bin-hadoop2.7
export SPARK_LOCAL_DIRS=/home/bigdata/spark-2.2.3-bin-hadoop2.7/local #配置spark的local目录
export SPARK_MASTER_IP=node01
export SPARK_LOCAL_IP=10.67.32.11
export SPARK_MASTER_WEBUI_PORT=8085 #web页面端口
export SPARK_WORKER_CORES=20 #Worker的cpu核数
export SPARK_WORKER_MEMORY=2g #worker内存大小
export SPARK_WORKER_DIR=/home/bigdata/spark-2.2.3-bin-hadoop2.7/worker #worker目录
export SPARK_WORKER_OPTS="-Dspark.worker.cleanup.enabled=true -Dspark.worker.cleanup.appDataTtl=604800" #worker自动清理及清理时间间隔
export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=18080 -Dspark.history.retainedApplications=3 -Dspark.history.fs.logDirectory=hdfs://node01:9000/spark/history" #history server页面端口>、备份数、log日志在HDFS的位置
export SPARK_LOG_DIR=/home/bigdata/spark-2.2.3-bin-hadoop2.7/logs
export SPARK_PID_DIR=/home/bigdata/spark-2.2.3-bin-hadoop2.7/pid
export SPARK_JAVA_OPTS="-Dfile.encoding=UTF-8 -Dsun.jnu.encoding=UTF-8"
```

### Spark 启动

```
/home/bigdata/spark-2.2.3-bin-hadoop2.7/sbin/start-all.sh
```

### Spark web管理

```
http://10.67.32.11:8085/
```

# Hbase部署

### 环境配置

### hbase-env.sh

```
export HBASE_CLASSPATH=/home/bigdata/hbase-2.1.9/lib
export HBASE_PID_DIR=/home/bigdata/hbase-2.1.9/data
export HBASE_LOG_DIR=/home/bigdata/hbase-2.1.9/logs
export HBASE_MANAGES_ZK=false
```

### hbase-site.xml

```
<configuration>
    <property>
      <name>hbase.tmp.dir</name>
      <value>/home/bigdata/hbase-2.1.9/data</value>
    </property>
    <property>
      <name>hbase.rootdir</name>
      <value>hdfs://node01:9000/hbase</value>
    </property>
    <property>
      <name>hbase.cluster.distributed</name>
      <value>true</value>
    </property>
    <property>
      <name>hbase.zookeeper.quorum</name>
      <value>node01</value>
    </property>
    <property>
      <name>hbase.zookeeper.property.clientPort</name>
      <value>2181</value>
    </property>
    <property>
      <name>hbase.zookeeper.property.dataDir</name>
      <value>/home/bigdata/zookeeper-3.4.8/data</value>
    </property>
    <property>
      <name>hbase.defaults.for.version.skip</name>
      <value>true</value>
    </property>
</configuration>
```

### 启动Hbase

```
/home/bigdata/hbase-2.1.9/bin/start-hbase.sh
```

### hbase管理

```
http://10.67.32.11:16010/
```

Hbase  命令行

```
./hbase shell
create 'student','Sname','Ssex','Sage','Sdept','course'
describe 'student'
list
put 'student','95001','Sname','LiYing'
get 'student','95001'
scan 'student'
delete 'student','95001','Ssex'
deleteall 'student','95001'
删除表，缺一不可
disable 'student'  
drop 'student'
```

# 机器学习平台搭建

### 卸载以前遗留

```
 rm -rf /etc/kubernetes/*
 rm -rf /etc/cni/net.d/
 yum remove -y kubelet kubeadm kubectl
```



### k8s安装

```
yum install -y kubelet-1.20.5 kubeadm-1.20.5 kubectl-1.20.5
```

### 安装其他组件

```
#!/bin/bash
images=(
kube-apiserver:v1.20.5
kube-proxy:v1.20.5
kube-controller-manager:v1.20.5
kube-scheduler:v1.20.5
coredns:1.7.0
etcd:3.4.13-0
pause:3.2
)
for imageName in ${images[@]}; do
  docker pull registry.cn-hangzhou.aliyuncs.com/google_containers/$imageName
done
```

### 添加网络配置

```
kubeadm init --pod-network-cidr=10.244.0.0/16 --image-repository=registry.cn-hangzhou.aliyuncs.com/google_containers
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.ym
```

# 安装kubeflow

### 安装kustomize

```
curl -Lo ./kustomize https://github.com/kubernetes-sigs/kustomize/releases/download/v3.2.0/kustomize_3.2.0_linux_amd64
chmod +x ./kustomize
sudo mv kustomize /usr/local/bin
```

### 安装kubeflow

```
wget https://github.com/kubeflow/manifests/archive/refs/tags/v1.5.1.tar.gz
tar -zxvf v1.5.1.tar.gz

```

# etcd

### 安装脚本

```
ETCD_VER=v3.4.15

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GITHUB_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /opt/soft/apisix/etcd

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /opt/soft/apisix/etcd --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

# install end and check etcd version
/opt/soft/apisix/etcd/etcd --version
/opt/soft/apisix/etcd/etcdctl version
```

### etcd.yaml 配置文件

```
name: etcd-1
data-dir: /opt/soft/apisix/etcd/data
listen-client-urls: http://10.67.32.12:12379
advertise-client-urls: http://10.67.32.12:12379
listen-peer-urls: http://10.67.32.12:12380
initial-advertise-peer-urls: http://10.67.32.12:12380
initial-cluster: etcd-1=http://10.67.32.12:12380

initial-cluster-token: etcd-cluster-token

initial-cluster-state: new
```

启动etcd

```
./etcd --config-file etcd.conf
```

# Apisix 安装测试

|     ip      | 端口 |                 备注                  |
| :---------: | :--: | :-----------------------------------: |
| 10.67.32.22 | 9000 | http://10.67.32.22:9000/ admin/admins |
|             |      |                                       |
|             |      |                                       |



### 安装

```
git clone https://github.com/apache/apisix-docker.git
cd apisix-docker/example
docker-compose -p docker-apisix up -d
```

### 创建带插件的路由

```
{
  "uri": "/*",
  "name": "test",
  "host": "httpbin1.org",
  "plugins": {
    "traffic-split": {
      "rules": [
        {
          "match": [
            {
              "vars": [
                [
                  "http_store_code",
                  "in",
                  [
                    "abx001",
                    "abx002"
                  ]
                ]
              ]
            }
          ],
          "weighted_upstreams": [
            {
              "upstream": {
                "name": "upstream_A",
                "nodes": {
                  "10.67.31.22:8822": 10
                },
                "type": "roundrobin"
              }
            }
          ]
        }
      ]
    }
  },
  "upstream": {
    "nodes": [
      {
        "host": "10.67.31.22",
        "port": 8833,
        "weight": 1
      }
    ],
    "timeout": {
      "connect": 6,
      "send": 6,
      "read": 6
    },
    "type": "roundrobin",
    "scheme": "http",
    "pass_host": "pass",
    "keepalive_pool": {
      "idle_timeout": 60,
      "requests": 1000,
      "size": 320
    }
  },
  "status": 1
}
```

### 灰度服务器-8822服务

```
from flask import Flask, request, jsonify
app = Flask(__name__)
@app.route('/get', methods=["GET", "POST"])
def calculate():
    if request.method == 'GET':
        params = request.args
    else:
        params = request.form if request.form else request.json
    a = params.get("a", 0)
    b = params.get("b", 0)
    c = int(a) + int(b)
    res = {"result": c}
    return jsonify(env='gray',
                   content=res)

if __name__ == '__main__':
    app.run(host='0.0.0.0',
            threaded=True,
            debug=False,
            port=8822)
```

### 生产服务器-8833 服务

```
from flask import Flask, request, jsonify
app = Flask(__name__)
@app.route('/get', methods=["GET", "POST"])
def calculate():
    if request.method == 'GET':
        params = request.args
    else:
        params = request.form if request.form else request.json
    a = params.get("a", 0)
    b = params.get("b", 0)
    c = int(a) + int(b)
    res = {"result": c}
    return jsonify(env='prod',
                   content=res)

if __name__ == '__main__':
    app.run(host='0.0.0.0',
            threaded=True,
            debug=False,
            port=8833)
```

### 请求头路由访问

```
curl "http://127.0.0.1:9080/get?a=1&b=4" -H "store_code: abx001" -H "Host: httpbin1.org" -i
curl "http://127.0.0.1:9080/get?a=1&b=4" -H "store_code: abx002" -H "Host: httpbin1.org" -i
```

etcd 命令

```
./etcdctl --endpoints=http://127.0.0.1:2379 get /apisix/routes/1
```

```
{"id":"1","create_time":1670997574,"update_time":1671012006,"uri":"/*","name":"test","priority":1,"host":"httpbin1.org","vars":[["http_store_code","==","abx004"]],"plugins":{"traffic-split":{"rules":[{"match":[{"vars":[["http_store_code","in",["abx001","abx002","abx003"]]]}],"weighted_upstreams":[{"upstream":{"name":"upstream_A","nodes":{"10.67.31.22:8822":10},"type":"roundrobin"}}]}]}},"upstream":{"nodes":{"10.67.31.22:8833":1},"timeout":{"connect":6,"send":6,"read":6},"type":"roundrobin","scheme":"http","pass_host":"pass","keepalive_pool":{"idle_timeout":60,"requests":1000,"size":320}},"status":1}
```

```
./etcdctl --endpoints=http://127.0.0.1:2379 put /apisix/routes/1 {"id":"1","create_time":1670997574,"update_time":1671012006,"uri":"/*","name":"test","priority":1,"host":"httpbin1.org","vars":[["http_store_code","==","abx004"]],"plugins":{"traffic-split":{"rules":[{"match":[{"vars":[["http_store_code","in",["abx001","abx002","abx003"]]]}],"weighted_upstreams":[{"upstream":{"name":"upstream_A","nodes":{"10.67.31.22:8822":10},"type":"roundrobin"}}]}]}},"upstream":{"nodes":{"10.67.31.22:8833":1},"timeout":{"connect":6,"send":6,"read":6},"type":"roundrobin","scheme":"http","pass_host":"pass","keepalive_pool":{"idle_timeout":60,"requests":1000,"size":320}},"status":1}
```



# 情感对话系统

|     Ip      | 端口 | 备注 |
| :---------: | :--: | :--: |
| 10.67.32.12 |      |      |
|             |      |      |
|             |      |      |

/opt/project/SATbot2.0

### 启动后台服务

```
conda activate nlp3.7.6
nohup gunicorn -b 0.0.0.0:5008 -t 600 "model:create_app()" &
or
gunicorn -b 0.0.0.0:5008 model.flask_backend_with_aws
```

### Vue 启动

```
npm run start
```

