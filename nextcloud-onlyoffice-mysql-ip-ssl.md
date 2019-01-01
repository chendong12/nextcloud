## 通过docker安装nextcloud＋onlyoffice＋mysql并配置`ip地址`ssl
## 主要内容
1. 通过docker安装nextcloud＋onlyoffice＋mysql
2. 配置通过https://192.168.9.51 登陆(**其中`192.168.9.51`为演示IP地址，你需要修改为你实际IP地址**)
3. 配置 SSL 自签名
4. 最终实现通过https://192.168.9.51 来访问服务器，避免出现ssl不可信的情况
## 系统要求
* 系统: Centos7
* 内存: 2G 以上
* 硬盘: 10G 以上
## 系统初始化
> 关闭selinux，自动更新时间，停止firewalld
```
setenforce 0
sed -i "s/SELINUX=enforcing/SELINUX=disabled/g" /etc/selinux/config
yum -y install ntp
service ntpd restart
cp -rf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
cd /root
echo '0-59/10 * * * * /usr/sbin/ntpdate -u cn.pool.ntp.org' >> /root/crontab.back
crontab /root/crontab.back
systemctl restart crond
yum install net-tools -y
yum install epel-release -y
systemctl stop firewalld
systemctl disable firewalld
yum install lynx wget expect iptables -y
```
## 1、安装程序
```
#安装 docker 相关程序
yum install git docker docker-compose -y
#运行docker
service docker start
#设置docker开机启动
systemctl enable docker
#安装nextcloud
cd /root/
git clone --recursive https://github.com/tvollscw/docker-onlyoffice-nextcloud-mysql
cd docker-onlyoffice-nextcloud-mysql
vi docker-compose.yml
#修改其中的 51、52行中的mysql密码
git submodule update --remote
#运行 Docker Compose:
docker-compose up -d
```
> 打开浏览器输入服务器IP地址，http://your_ip_address 在出现的配置页面中输入
> * 用户名：admin，管理密码
> * 数据库选择mysql，数据库的帐号、密码、数据库名、数据库地址信息，见docker-compose.yml 文件
> * 注意：配置的数据库地址localhost 为mariadb
***
> 配置完成之后，开启onlyoffice支持
```
cd /root/docker-onlyoffice-nextcloud-mysql/
bash set_configuration.sh
```
> 打开浏览器http://your_ip_address  通过上面配置的admin帐号和密码进行登陆
> **如果不需要通过域名访问，或者https访问，到此配置成功，您可以使用了**

### 文件位置说明
* nginx配置文件
/root/docker-onlyoffice-nextcloud-mysql/nginx.conf
* nextcloud数据存储位置
/var/lib/docker/volumes/dockeronlyofficenextcloudmysql_app_data/_data

## 制作自签名证书
```
docker exec -it app-server bash
mkdir cert
openssl req -new -x509 -days 365 -nodes -out ./cert/nextcloud.crt -keyout ./cert/nextcloud.key
```
> 会出现如下提示,根据提示输入省份、组织、邮箱地址等
```
Generating a RSA private key
...................................+++++
................................+++++
writing new private key to './cert/nextcloud.key'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:cn
State or Province Name (full name) [Some-State]:bj
Locality Name (eg, city) []:bj
Organization Name (eg, company) [Internet Widgits Pty Ltd]:ramu
Organizational Unit Name (eg, section) []:ramu
Common Name (e.g. server FQDN or YOUR name) []:ramu
Email Address []:zcm8483@gmail.com
```
> 退出 app-server docker ，并拷贝生成的证书到本地
```
exit
cd /root/docker-onlyoffice-nextcloud-mysql/
mkdir cert
docker container cp app-server:/var/www/html/cert ./
```

* 修改docker-compose 文件，映射本地证书文件夹到docker 的nginx下
```
cd /root/docker-onlyoffice-nextcloud-mysql/
vi docker-compose.yml
```
* 在nginx 的volumes 下面增加目录映射，如下所示最后一行
```
nginx:
     volumes:
      - ./cert:/etc/nginx/cert
```
* 修改nginx 配置文件，配置ssl证书
```
vi /root/docker-onlyoffice-nextcloud-mysql/nginx.conf
```
>  （1）在server部分做如下改动，ssl 开头的三行内容是增加的
```
  server {
        listen 443 ssl;
        server_name 192.168.9.51;
        ssl_certificate /etc/nginx/cert/nextcloud.crt;
        ssl_certificate_key /etc/nginx/cert/nextcloud.key;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
```

> （2）然后在倒数第二行增加如下内容目的是为了访问http 的时候自动跳转到https
```
server {
           listen 80;
           server_name 192.168.9.51;
           return 301 https://$host$request_uri;
      }
```

* 通过浏览器登陆 nextcloud 修改onlyoffice 配置
> 到设置中选择onlyoffice，打开onlyoffice api 页面
> 把Server address for internet requests form the Document Editing Src，修改为你的nextcloud 访问地址https://192.168.9.51

* 修改 nextcloud 配置文件
```
vi /var/lib/docker/volumes/dockeronlyofficenextcloudmysql_app_data/_data/config/config.php
```
> 修改其中的30、43行为你的nextcloud 访问地址https://192.168.9.51

## 修改onlyoffice 的default.json 文件，信任不可信的ssl
```
docker container cp onlyoffice-document-server:/etc/onlyoffice/documentserver/default.json ./
vi default.json
```
> 修改"rejectUnauthorized": true 为"rejectUnauthorized": false
```
vi docker-compose.yml 
```
> 在onlyoffice-document-server 的volumes 部分增加最后一行
```
onlyoffice-document-server:
    container_name: onlyoffice-document-server
    image: onlyoffice/documentserver:latest
    stdin_open: true
    tty: true
    restart: always
    networks:
      - onlyoffice
    expose:
      - '80'
      - '443'
    volumes:
      - document_data:/var/www/onlyoffice/Data
      - document_log:/var/log/onlyoffice
      - ./default.json:/etc/onlyoffice/documentserver/default.json
```
* 重启docker-compose
```
cd /root/docker-onlyoffice-nextcloud-mysql/
docker-compose down
docker-compose up -d
```
**注意:用docker-compose restart 是不能加载分区的 **
