## 通过docker安装nextcloud＋onlyoffice＋mysql
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
