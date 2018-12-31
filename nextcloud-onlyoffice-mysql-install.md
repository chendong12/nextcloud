#           通过docker安装nextcloud＋onlyoffice＋mysql
## 主要内容
. 通过docker安装nextcloud＋onlyoffice＋mysql
. 配置通过域名登陆
* 3、通过letsencrypt 配置免费 SSL 证书
* 4、最终实现通过https://cloud.rexen.ne 来访问服务器，避免出现ssl不可信的情况
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
>> 用户名：admin，管理密码
>> 数据库选择mysql，数据库的帐号、密码、数据库名、数据库地址信息，见docker-compose.yml 文件
> 注意：配置的数据库地址localhost 为mariadb
> 配置完成之后，开启onlyoffice支持
```
cd /root/docker-onlyoffice-nextcloud-mysql/
bash set_configuration.sh
```
> 打开浏览器http://your_ip_address  通过上面配置的admin帐号和密码进行登陆
**如果不需要通过域名访问，或者https访问，到此配置成功，您可以使用了**

### 文件位置说明
* nginx配置文件
/root/docker-onlyoffice-nextcloud-mysql/nginx.conf
* nextcloud数据存储位置
/var/lib/docker/volumes/dockeronlyofficenextcloudmysql_app_data/_data

## 2、配置通过域名访问nextcloud
* 1）修改 nginx 配置文件
```
vi /root/docker-onlyoffice-nextcloud-mysql/nginx.conf
```
> 在server 部分增加下面的内容，后面的cloud.rexen.net 为你的域名
```
server_name cloud.rexen.net;
```
* 2）修改 nextcloud 配置文件
```
vi /var/lib/docker/volumes/dockeronlyofficenextcloudmysql_app_data/_data/config/config.php
```
> 修改24、30行的IP地址为域名
```
array (
    0 => 'cloud.rexen.net',
    1 => 'nginx-server',
  ),
  'datadirectory' => '/var/www/html/data',
  'dbtype' => 'mysql',
  'version' => '15.0.0.10',
  'overwrite.cli.url' => 'http://cloud.rexen.net',

```
## 通过letsencrypt 配置免费 SSL 证书
* 停止 nextcloud 
```
cd /root/docker-onlyoffice-nextcloud-mysql/
docker-compose stop
```
* 拉取并运行镜像，生成证书
```
docker pull quay.io/letsencrypt/letsencrypt:latest
docker run -it --rm -p 80:80 -p 443:443  -v /etc/letsencrypt:/etc/letsencrypt quay.io/letsencrypt/letsencrypt auth
```
* 选择第一项，然后输入域名、邮箱地址等信息，最后会生成证书文件

```
Warning: This Docker image will soon be switching to Alpine Linux.
You can switch now using the certbot/certbot repo on Docker Hub.
/opt/certbot/venv/local/lib/python2.7/site-packages/cryptography/hazmat/primitives/constant_time.py:26: CryptographyDeprecationWarning: Support for your Python version is deprecated. The next version of cryptography will remove support. Please upgrade to a 2.7.x release that supports hmac.compare_digest as soon as possible.
  utils.DeprecatedIn23,
Saving debug log to /var/log/letsencrypt/letsencrypt.log

How would you like to authenticate with the ACME CA?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: Spin up a temporary webserver (standalone)
2: Place files in webroot directory (webroot)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 1
Plugins selected: Authenticator standalone, Installer None
Please enter in your domain name(s) (comma and/or space separated)  (Enter 'c'
to cancel): cloud.rexen.net
Obtaining a new certificate
Performing the following challenges:
http-01 challenge for cloud.rexen.net
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/cloud.rexen.net/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/cloud.rexen.net/privkey.pem
   Your cert will expire on 2019-03-29. To obtain a new or tweaked
   version of this certificate in the future, simply run certbot
   again. To non-interactively renew *all* of your certificates, run
   "certbot renew"
 - If you like Certbot, please consider supporting our work by:

   Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
   Donating to EFF:                    https://eff.org/donate-le
```
* 复制上面生成的证书文件
```
mkdir /root/docker-onlyoffice-nextcloud-mysql/cert
cp  /etc/letsencrypt/live/cloud.rexen.net/fullchain.pem /root/docker-onlyoffice-nextcloud-mysql/cert
cp /etc/letsencrypt/live/cloud.rexen.net/privkey.pem /root/docker-onlyoffice-nextcloud-mysql/cert
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
```vi /root/docker-onlyoffice-nextcloud-mysql/nginx.conf
```
>  （1）在server部分做如下改动，ssl 开头的三行内容是增加的
```
  server {
        listen 443 ssl;
        server_name cloud.rexen.net;
        ssl_certificate /etc/nginx/cert/fullchain.pem;
        ssl_certificate_key /etc/nginx/cert/privkey.pem;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
```

> （2）然后在倒数第二行增加如下内容目的是为了访问http 的时候自动跳转到https
```
server {
           listen 80;
           server_name cloud.rexen.net;
           return 301 https://$host$request_uri;
      }
```

* 通过浏览器登陆 nextcloud 修改onlyoffice 配置
> 到设置中选择onlyoffice，打开onlyoffice api 页面
>> 把Server address for internet requests form the Document Editing Src，修改为你的https的域名，我的https域名为 https://cloud.rexen.net
* 修改 nextcloud 配置文件
```
vi /var/lib/docker/volumes/dockeronlyofficenextcloudmysql_app_data/_data/config/config.php
```
> 修改其中的30、43行为你的https域名
* 重启docker-compose
```
cd /root/docker-onlyoffice-nextcloud-mysql/
docker-compose down
docker-compose up -d
```
** 注意：用docker-compose restart 是不能加载分区的 **
