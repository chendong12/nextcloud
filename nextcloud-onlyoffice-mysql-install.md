#            通过docker安装nextcloud＋onlyoffice＋mysql
## 系统要求
本教程是在Centos7 上部署的
## 系统初始化
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
打开浏览器输入服务器IP地址，http://your_ip_address 在出现的配置页面中输入
用户名：admin，管理密码
数据库选择mysql，数据库的帐号、密码、数据库名、数据库地址信息，见docker-compose.yml 文件
注意：配置的数据库地址localhost 为mariadb
配置完成之后，开启onlyoffice支持
```
cd /root/docker-onlyoffice-nextcloud-mysql/
bash set_configuration.sh
```
打开浏览器http://your_ip_address  通过上面配置的admin帐号和密码进行登陆
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
在server 部分增加下面的内容，后面的cloud.rexen.net 为你的域名
```
server_name cloud.rexen.net;
```
* 2）修改 nextcloud 配置文件
```
vi /var/lib/docker/volumes/dockeronlyofficenextcloudmysql_app_data/_data/config/config.php
```

```
array (
    0 => '`cloud.rexen.net`',
    1 => 'nginx-server',
  ),
  'datadirectory' => '/var/www/html/data',
  'dbtype' => 'mysql',
  'version' => '15.0.0.10',
  'overwrite.cli.url' => 'http://cloud.rexen.net',

```


