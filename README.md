基于LNMP的个人博客网站搭建
项目描述： 在本地虚拟机（VMware）中部署一套完整的LNMP环境，并成功安装和运行WordPress博客程序，理解Web请求的完整流程。

操作系统： CentOS 7.x
服务组件：
  L: Linux (CentOS)
  N: Nginx (作为Web服务器和反向代理)
  M: MySQL/MariaDB (作为数据库)
  P: PHP (作为动态语言处理器）

阶段一：系统准备与初始化

1. 系统安装与登录
   使用VMware安装一台CentOS 7虚拟机。
   使用 ssh root@你的服务器IP 远程连接到虚拟机。
2. 更新系统并关闭防火墙/SELinux

一·关闭  firewalld  防火墙

1. 临时关闭：
执行以下命令，临时关闭  firewalld ，系统重启后会恢复：
sudo systemctl stop firewalld
2. 永久关闭：
若要永久关闭（系统重启后也不启动），执行：
sudo systemctl disable firewalld

二·关闭 SELinux

1. 临时关闭：
执行命令临时将 SELinux 设为  permissive  模式（仅警告，不强制限制）：
sudo setenforce 0
 2. 永久关闭：
编辑  /etc/selinux/config  文件，将  SELINUX=enforcing  改为  SELINUX=disabled ，然后保存文件。之后需要重启系统，修改才能生效：
sudo vi /etc/selinux/config
 （在文件中找到  SELINUX=enforcing  这一行，替换为  SELINUX=disabled ）

三·CentOS7官方yum源停服，替换为阿里源

备份

mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
mv /etc/yum.repos.d/epel.repo /etc/yum.repos.d/epel.repo.backup
mv /etc/yum.repos.d/epel-testing.repo /etc/yum.repos.d/epel-testing.repo.backup
CentOS 7

wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
epel(RHEL 7)

wget -O /etc/yum.repos.d/epel.repo https://mirrors.aliyun.com/repo/epel-7.repo
清理并生成新的缓存

sudo yum clean all
sudo yum makeache
在/目录下 mkdir /www
阶段二：分步安装LNMP组件

1.安装Nginx

设置Nginx的YUM仓库

vim /etc/yum.reops.d/nginx.reop

[nginx-stable] 
name=nginx stable repo 
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/ 
gpgcheck=1 
enabled=1 
gpgkey=https://nginx.org/keys/nginx_signing.key 
module_hotfixes=true
安装Nginx
   yum install nginx -y
   启动Nginx并设置开机自启
   systemctl start nginx
   systemctl enable nginx

验证： 在物理机浏览器输入虚拟机的IP地址，看到 “Welcome to nginx…” 页面即成功。

2.安装MYSQL(MariaDB)

安装MariaDB（MySQL的一个分支，完全兼容）
   yum install mariadb-server mariadb -y
   启动并设置开机自启
   systemctl start mariadb
   systemctl enable mariadb

执行安全配置脚本，设置root密码等: mysql_secure_installation
在执行 mysql_secure_installation 时，会提示你：
     · 设置 root 密码。
     · 移除匿名用户。
     · 禁止 root 远程登录。
     · 移除 test 数据库。
     · 重新加载权限表。

3. 安装 PHP
   由于Nginx不直接处理PHP，我们需要安装PHP-FPM（FastCGI Process Manager）
   需要安装更丰富的PHP扩展包，以支持WordPress
   yum install php php-fpm php-mysqlnd php-gd php-xml php-mbstring -y
   启动PHP-FPM并设置开机自启
   systemctl start php-fpm
   systemctl enable php-fpm

注：WordPress 对 PHP 版本有要求（推荐 PHP 7.4 及以上版本），较低的 PHP 版本可能会导致 WordPress 安装或运行出现问题。

升级 PHP 版本

首先，需要添加合适的 PHP 软件源。以 CentOS 系统为例，可使用 Remi 源来安装高版本 PHP。
安装 Remi 源：
sudo yum install -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm
 

启用 PHP 7.4 源：
sudo yum-config-manager –enable remi-php74
 

然后，安装 PHP 7.4 及相关扩展（WordPress 所需的常见扩展，如  php – mysqlnd 、 php – gd 、 php – xml  等）：
bash
sudo yum install -y php php – mysqlnd php – gd php – xml php – mbstring
 

安装完成后，查看 PHP 版本，确认升级成功：
php -v  
重启 Web 服务器
升级 PHP 后，需要重启 Nginx（或 Apache）服务，使新的 PHP 配置生效。以 Nginx 为例：sudo systemctl restart nginx
阶段三：配置服务，使其协同工作

1. 配置 Nginx 以支持 PHP
   · 编辑Nginx的默认配置文件：
   vim /etc/nginx/nginx.conf
   · 将其修改为如下内容（关键步骤！）：
    server {
       listen       80;
       server_name  localhost; # 或你的服务器IP

       # 网站根目录，我们放在 /www
       root   /www;
       index  index.php index.html index.htm;

       location / {
           try_files $uri $uri/ =404;
       }

       # 关键配置：将所有以.php结尾的请求交给PHP-FPM处理
       location ~ \.php$ {
           try_files $uri =404;
           fastcgi_pass   127.0.0.1:9000; # PHP-FPM默认监听9000端口
           fastcgi_index  index.php;
           fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
           include        fastcgi_params;
       }
   }
    · 保存退出后，检查配置语法并重启Nginx。
nginx -t # 检查语法，看到 “successful” 即可
   systemctl restart nginx

2. 测试 PHP 与 Nginx 的协作
   · 创建一个PHP信息文件来测试：

vim /www/info.php
  #！/bin/bash
   echo “<?php phpinfo(); ?>” > /www/info.php
   · 在浏览器访问 http://你的服务器IP/info.php。
   · 成功标志： 看到一个显示PHP版本和配置详情的页面。这说明Nginx已经能够正确解析PHP文件。

阶段四：部署 WordPress

1. 为WordPress创建数据库和用户
   # 登录MySQL
   mysql -u root -p
   · 在MySQL命令行中执行：
  sql
   — 创建一个名为`wordpress`的数据库
   CREATE DATABASE wordpress;
   — 创建一个用户`wpuser`，并设置密码（例如：`wppassword`）
   CREATE USER ‘wpuser’@’localhost’ IDENTIFIED BY ‘wppassword’;
   — 授予wpuser用户对wordpress数据库的所有权限
   GRANT ALL PRIVILEGES ON wordpress.* TO 'wpuser'@'localhost';
   -- 刷新权限
   FLUSH PRIVILEGES;
   -- 退出
   EXIT;
2. 下载并配置 WordPress
   # 进入网站根目录
   cd /www
   # 下载最新版WordPress（请去官网复制最新的.tar.gz链接）
   wget https://wordpress.org/latest.tar.gz
   # 解压
   tar -xzvf latest.tar.gz
   # 复制WordPress配置文件样本
   cd wordpress
   cp wp-config-sample.php wp-config.php
   # 编辑配置文件，填入数据库信息
   vim wp-config.php
   · 在 wp-config.php 中找到并修改以下行：
   php
   // ** MySQL 设置 – 具体信息来自您刚才提供的内容 ** //
   define( ‘DB_NAME’, ‘wordpress’ );
   define( ‘DB_USER’, ‘wpuser’ );
   define( ‘DB_PASSWORD’, ‘wppassword’ );
   define( ‘DB_HOST’, ‘localhost’ );

3. 设置权限并完成安装
   # 将WordPress目录的所有权给Nginx用户（通常是nginx或apache）
   chown -R nginx:nginx /www/wordpress/
   # 重启所有相关服务
   systemctl restart nginx php-fpm mariadb
   · 最终验证： 在浏览器访问 http://你的服务器IP。· 成功标志： 你看到了WordPress著名的“五分钟安装页面”，按照提示设置站点标题、管理员账号和密码，即可完成安装并登录到你的博客后台！
