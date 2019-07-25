#### 1. 配置文件路径
```
/etc/httpd/conf/httpd.conf 
```
#### 2. 添加多虚拟主机
```
Listen 80//监听端口，如果是阿里云的话还要在服务器的白名单打开80端口权限。
<VirtualHost *:80> #第一个主机，80端口
     DocumentRoot "/root/abc" #指向本地位置
     ServerName www.abc.com #主机名称（注意这个很重要，就是你的域名，准确输入才能成功）
</VirtualHost> #结束第一个主机配置
<VirtualHost *:80> #第二个主机，80端口
     DocumentRoot "/root/def" #指向本地位置
     ServerName www.def.com #主机名称
</VirtualHost>
//如果增加更多的虚拟机可以像上边一样，添加多个。
每一个虚拟机都可以设置自己的目录权限。
```
### 3.重新启动Apache
```
service httpd restart//根据自己装的版本启动 Apache
```
