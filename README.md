# huaweicloud-solution-GameFlexMatch

# 简介

`GameFlexMatch`是一个服务托管解决方案，包含四个服务组件(`Fleetmanager`/`AppGateway`/`AASS`/`AuxProxy`)，可以实现应用的托管、托管应用所需资源的弹性伸缩、应用进程的资源调度管理、应用的灰度发布，多`region`部署时可以实现用户的就近接入，减少时延，以及服务资源的跨地域容灾。可以帮助开发者快速构建稳定、低延时的多人游戏的部署环境，并节省大量的运维成本，支持`Unreal`、`Unity`引擎，`C#`、`C++`以及`gRPC`支持的任何语言的`server`框架部署和运行。


# 逻辑架构
<img src="img/architecture.jpg" width="80%">

GameFlexMatch平台由五个服务组件组成：

+ [FleetManager](https://gitee.com/HuaweiCloudDeveloper/huaweicloud-solution-gameflexmatch-fleetmanager): 负责应用进程的全局化动态部署及管理，支持配置动态部署策略，基于成本或时延优化应用分布，负责弹性伸缩策略的配置和服务端会话、客户端会话与应用包的管理，服务端应用的灰度发布等
+ [AppGateway](https://gitee.com/HuaweiCloudDeveloper/huaweicloud-solution-gameflexmatch-appgateway): 负责应用进程、会话与客户端连接的管理，通过与`AuxProxy`通信获得应用进程信息，决策进程资源的调度
+ [AASS](https://gitee.com/HuaweiCloudDeveloper/huaweicloud-solution-gameflexmatch-aass): 负责弹性伸缩组和弹性伸缩策略的管理与执行，以及服务端应用资源的监控，调用华为云`AS`(弹性伸缩服务)实现资源的弹性伸缩
+ [AuxProxy](https://gitee.com/HuaweiCloudDeveloper/huaweicloud-solution-gameflexmatch-auxproxy): 在扩容出的实例中自动拉起，负责应用进程的创建、进程状态的上报以及应用进程的通信
+ [Console](https://gitee.com/HuaweiCloudDeveloper/huaweicloud-solution-gameflexmatch-console): 运维平台，用于监控`GameFlexMatch`的运行状态，以及运维管理`GameFlexMatch`的`fleet`、应用包与用户信息等



# 创建资源

### 创建ECS

购买4台ECS，分别部署GameFlexMatch服务组件，并将管理面资源部署在同一VPC下，以下测试规格，具体规格按需选择：

|        | 资源规格   | 服务组件     | 监听端口 | 是否必须配置EIP |
| ------ | ---------- | ------------ | -------- | --------------- |
| ECS-01 | 2vCPUs/4GB | appgateway   | 60003    | Y               |
| ECS-02 | 2vCPUs/4GB | aass         | 9091     | Y               |
| ECS-03 | 2vCPUs/4GB | fleetmanager | 31002    | Y               |
| ECS-04 | 2vCPUs/4GB | console      | 80       | Y               |

安全组规则中入方向增加ECS的VPC网段，以便服务组件互相通信



### 创建数据库

购买华为云RDS快速构建，按需选择规格，默认端口3306，创建appgateway，aass和fleetmanager三个数据库。





### 创建Redis

购买Redis服务组件，配置访问端口和访问密码，三个服务组件共用一个Redis数据库。



### 创建InfluxDB

购买GaussDB(for Influx)，并开启SSL安全连接，使用默认证书即可，为服务组件创建数据库(aass/appgateway)，aass与appgateway共用一个influxDB的数据库。



### 创建密钥对

在数据加密服务DEW中，密钥对管理中，创建私有密钥对（仅限该IAM账号使用）或账号密钥对（推荐，其他账户也可使用），用于登录弹性伸缩实例。



# 服务安装

## 前置条件

- 操作系统：CentOS 7.6 64bit

- 实例最低配置：2U4G

- 安装git

  ```shell
  yum -y install git
  ```

  

## 前端组件

### 环境安装

#### 安装 nodejs


```shell
wget https://nodejs.org/download/release/v16.13.1/node-v16.13.1-linux-x64.tar.gz
tar xf node-v16.13.1-linux-x64.tar.gz
mv node-v16.13.1-linux-x64 /usr/local/
```

新建并修改 node.sh 文件

```shell
vi /etc/profile.d/node.sh
```

设置环境变量

```shell
export NODE_HOME=/usr/local/node-v16.13.1-linux-x64
export PATH=${NODE_HOME}/bin:$PATH
```

执行脚本使环境变量生效

```shell
chmod +x /etc/profile.d/node.sh
source /etc/profile.d/node.sh
```

#### 安装 vue

```shell
npm install -g @vue/cli
```

#### 安装编译工具及库文件

```shell
yum -y install make zlib zlib-devel gcc-c++ libtool  openssl openssl-devel
```

下载 PCRE 安装包

```shell
cd /usr/local/src/ || exit
wget http://downloads.sourceforge.net/project/pcre/pcre/8.35/pcre-8.35.tar.gz
tar zxvf pcre-8.35.tar.gz
```

编译安装

```shell
cd pcre-8.35 || exit
./configure
make && make install
```

#### 安装 Nginx

```shell
cd /usr/local/src/ || exit
wget http://nginx.org/download/nginx-1.7.8.tar.gz
tar zxvf nginx-1.7.8.tar.gz
cd nginx-1.7.8 || exit
```

编译安装到/usr/local/webserver/nginx

```shell
./configure --prefix=/usr/local/webserver/nginx --with-http_stub_status_module --with-http_ssl_module --with-pcre=/usr/local/src/pcre-8.35
make
make install
```

替换 /usr/local/webserver/nginx/conf/nginx.conf 为以下内容，替换proxy_pass的地址为后端服务器IP

```nginx
worker_processes  1;

events {
    worker_connections  1024;
}

http {
    include       mime.types;
    default_type  application/octet-stream;
    sendfile        on;
    keepalive_timeout  300;
    client_max_body_size 6g;
    server {
        listen       80;
        server_name  localhost;

        location /api/ {
            # 修改以下IP为后端服务器IP，把 /api 路径下的请求转发给真正的后端服务器
            proxy_pass https://127.0.0.1:8080/;
        }

        location / {
                root html;
                try_files $uri /index.html;  # try_files：检查文件； $uri：监测的文件路径； /index.html：文件不存在重定向的新路径
                index index.html;
        }

        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
}

```



### 打包部署

拉取前端代码

```shell
cd /usr/local
git clone -b master-dev https://gitee.com/HuaweiCloudDeveloper/huaweicloud-solution-gameflexmatch-console.git
cd huaweicloud-solution-gameflexmatch-console
```

修改密码加密公钥配置文件 代码根目录/src/api/crypto.ts 第 42 行

```shell
export function encryptedData(data: string) {
    const encryptor = new JSEncrypt()
    let key = `XXXXXXXXX` // 设置公钥
    encryptor.setPublicKey(key)
    return encryptor.encrypt(data).toString()
}
```

在代码根目录下运行 npm install 命令，安装项目所需要的依赖

```shell
npm install
```

在根目录下安装 vue 国际化插件

```shell
npm install --save vue-i18n@next
```

在代码根目录下运行 npm run build 命令，将项目编译打包至根目录的 dist 文件夹下。

```shell
npm run build
```



把 dist 目录下的所有文件都复制到 nginx 网站根目录 /usr/local/webserver/nginx/html 下

配置 nginx 开机自启动，在/lib/systemd/system/目录下创建 nginx.service 文件

```shell
vi /lib/systemd/system/nginx.service
```

在该文件中添加如下内容

```
[Unit]
Description=nginx service
After=network.target
[Service]
Type=forking
ExecStart=/usr/local/webserver/nginx/sbin/nginx
ExecReload=/usr/local/webserver/nginx/sbin/nginx -s reload
ExecStop=/usr/local/webserver/nginx/sbin/nginx -s quit
PrivateTmp=true
[Install]
WantedBy=multi-user.target
```

设置文件的执行权限

```shell
chmod a+x /lib/systemd/system/nginx.service
```

设置开机自启动

```shell
systemctl enable nginx.service
```

启动nginx服务 

```shell
systemctl start nginx.service
```

浏览器输入该ECS绑定的ip地址即可访问GameFlexMatch前端界面，或输入: http://{ipv4}:80





## 后端组件

### 环境安装

- 版本要求：go1.16及以上版本
- 安装go

```shell
yum install golang
# 设置编译的可执行文件的操作系统
go env -w GOOS=linux
# 配置go代理
go env -w GO111MODULE=on
go env -w GOPROXY=https://repo.huaweicloud.com/repository/goproxy/
go env -w GONOSUMDB=*
```



### 文件编译

将源码下载到本地，编译`linux`可执行的二进制文件，**以下步骤中{version}中的变量需按具体情况更改**

```shell
# 1. fleetmanager
cd ~/huaweicloud-solution-gameflexmatch-fleetmanager
# 下载依赖包
go mod tidy
go build ./main.go
# 修改文件名
mv main fleetmanager-{version}

# 2. appgateway
cd ~/huaweicloud-solution-gameflexmatch-appgateway
go mod tidy
go build ./cmd/application_gateway.go
# 修改文件名
mv application_gateway appgateway-{version}

# 3. aass
cd ~/huaweicloud-solution-gameflexmatch-aass
go mod tidy
go build ./cmd/application-auto-scaling-service/application_auto_scaling_service.go
# 修改文件名
mv application_auto_scaling_service aass-{version}

# 4. auxproxy
cd ~/huaweicloud-solution-gameflexmatch-auxproxy
go mod tidy
go build ./cmd/auxproxy.go
```
### **证书准备**

在`linux`系统下通过`openssl`获取自签名证书，可在任一台`ECS`下操作，三个服务组件使用相同的自签名证书：

+ 获取`https`签名证书
```sh
# 1. 创建tlsSecret文件夹
mkdir -p /home/tlsSecret
cd /home/tlsSecret
# 2. 生成私钥tls.key
openssl genrsa -out tls.key 3072
# 3. 使用私钥生成csr，并查看
openssl req -new -key tls.key -out tls.csr
# 上述步骤会要求输入以下信息，可按实际情况填写，如
# There are quite a few fields but you can leave some blank
# For some fields there will be a default value,
# If you enter '.', the field will be left blank.

# Country Name (2 letter code) [XX]:China
# State or Province Name (full name) []:GuangDong
# Locality Name (eg, city) [Default City]:ShenZhen
# Organization Name (eg, company) [Default Company Ltd]:Huawei
# Organizational Unit Name (eg, section) []:Cloud
# Common Name (eg, your name or your server's hostname) []:GameFlexMatch
# Email Address []:GameFlexMatch@huawei.com

# Please enter the following 'extra' attributes
# to be sent with your certificate request
# A challenge password []:********
# An optional company name []:Huawei

openssl req -in tls.csr -text
# 4. 生成自签名证书 tls.crt, 并查看
openssl x509 -req -days 365 -in tls.csr -signkey tls.key -out tls.crt
openssl x509 -in tls.crt -text
```
+ 获取网络传输的RSA非对称加密的公钥与私钥，用户敏感数据如登录密码以及云资源的加密与解密
```sh
# 1. 创建RSA私钥，长度可以为1024，也可以为2048
cd /home/tlsSecret
openssl genrsa -out rsa_private.pem 2048
# 2. 在私钥的基础上生成公钥
openssl rsa -in rsa_private.pem -pubout -out rsa_public.pem
```

### **服务组件安装**

#### 安装appgateway服务组件

```sh
# 1. 登录ECS-01，新建/home/tlsSecret，并上传已生成的tls.crt与tls.key，以及RSA非对称加密的公钥与私钥rsa_private.pem，rsa_public.pem
mkdir -p /home/tlsSecret
# 2. 创建文件夹/home/appgateway/conf/hmac，
mkdir -p /home/appgateway/conf/hmac
# 上传client_hmac_conf.json、server_hmac_conf.json上传至hmac文件夹，样例见: 

# doc/build/appgateway/client_hmac_conf.json
# doc/build/appgateway/server_hmac_conf.json

# 3. 新建bin目录，上传可执行二进制文件与启动脚本并修改权限
mkdir -p /home/appgateway/bin
cd /home/appgateway/bin
# 上传生成的appgateway-{version}二进制文件与启动脚本appgateway_run.sh，并修改相关配置(样例见：doc/build/appgateway/appgateway_run.sh)
# 修改权限
chmod 750 appgateway-{version}
chmod 750 appgateway_run.sh

# 4. 配置完成后运行启动脚本
./appgateway_run.sh

# 5. 验证是否执行成功
ps -aux | grep appgateway

# 6. 关掉进程后配置开机自启动
# 在/etc/systemd/system下新建appgateway.service
# appgateway.service 样例见 /doc/build/appgateway
# 启动appgateway.service保证进程自动拉起
systemctl enable appgateway.service
systemctl start appgateway.service

# 7. 验证是否成功
ps -aux | grep appgateway
# 8. 可以正常运行则为部署成功

```

#### 安装aass服务组件

```sh
# 1. 登录ECS-02，并创建tlsSecret文件夹
mkdir -p /home/tlsSecret
# 2. 将已生成的tls.crt与tls.key上传至tlsSecret文件夹中
# 3. 创建文件夹/home/aass
mkdir -p /home/aass/configmap
# 4. 将server_hmac_conf.json与service_config.json上传至configmap中,修改相关配置，样例见：
# doc/build/aass/server_hmac_conf.json
# doc/build/aass/service_config.json

# 5. 创建文件夹/home/aass/bin
mkdir -p /home/aass/bin
# 6. 将aass的二进制可执行文件aass-{version}与执行脚本aass_run.sh上传至bin目录下，修改相关配置，并修改文件权限
cd /home/aass/bin
chmod 750 aass-{version}
chmod 750 aass_run.sh

# 7. 执行并验证是否执行成功
sh ./aass_run.sh
ps -aux | grep aass

# 8. Ctrl+c 关掉进程后配置开机自启动
# 在/etc/systemd/system下新建aass.service
# aass.service 样例见 /doc/build/aass
# 启动aass.service保证进程自动拉起
systemctl enable aass.service
systemctl start aass.service

# 9. 验证是否成功
ps -aux | grep aass
# 10. 可以正常运行则为部署成功

```

#### 安装fleetmanager服务组件

```sh
# 1. 登录ECS-03，创建文件夹tlsSecret并上传已生成的tls.crt与tls.key文件
mkdir -p /home/tlsSecret
# 2. 创建/home/configmap文件夹
mkdir -p /home/fleetmanager/configmap
# 3. 上传server_config.json到configmap下，并修改相关配置
# 样例见 doc/build/fleetmanager/server_config.json

# 4. 创建文件夹/home/fleetmanager/bin/conf/workflow
mkdir -p /home/fleetmanager/bin/conf/workflow

# 5. 上传create_fleet_workflow.json、delete_fleet_workflow.json以及create_build_image_workflow.json，详见
# doc/build/fleetmanager/create_fleet_workflow.json
# doc/build/fleetmanager/delete_fleet_workflow.json
# doc/build/fleetmanager/create_build_image_workflow.json

# 6. 上传fleetmanager的二进制可执行文件fleetmanager-{version}
# 与启动脚本fleetmanager_run.sh上传至bin文件夹，修改相关配置与文件权限
cd /home/fleetmanager/bin
chmod 750 fleetmanager-{version}
chmod 750 fleetmanager_run.sh

# 7. 启动脚本并验证是否成功
sh ./fleetmanager_run.sh
ps -aux | grep fleetmanager

# 8. 关掉进程后配置开机自启动
# 在/etc/systemd/system下新建fleetmanager.service
# fleetmanager.service 样例见 /doc/build/fleetmanager
# 启动fleetmanager.service保证进程自动拉起
systemctl enable fleetmanager.service
systemctl start fleetmanager.service

# 9. 验证是否成功
ps -aux | grep fleetmanager
# 10. 可以正常运行则为部署成功
```

## **其他说明**

   + 前端部署指导详见 [doc/build/console.md](/doc/build/console.md)
   + 应用镜像制作详见 [doc/build/make-image-guide.md](/doc/build/make-image-guide.md)
   + 部署过程中必要的参数注解详见 [doc/build/param-annotation.md](/doc/build/param-annotation.md)
   + 平台用户管理模块使用详见 [doc/build/user-management.md](/doc/build/user-management.md)
   + console平台的用户指南详见`/doc/user-guide`
   + 版本更新记录详见`doc/version/`

## 辅助工具
1. 加密工具：提供了GCM与RSA加解密敏感数据的工具，详见`/tools/cipher`
2. 对等连接工具：提供了两个`VPC`创建对等连接的工具，详见`/tools/vpc-peering`