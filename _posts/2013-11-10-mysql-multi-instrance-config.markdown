---
layout: post
status: publish
published: true
title: MYSQL多实例配置、使用
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
excerpt: "在实际的开发过程中，可能会需要在一台服务器上部署多个MYSQL实例，那建议使用MYSQL官方的解决方案 mysqld_multi\r\n\r\n&nbsp;\r\n\r\n<strong>1.修改my.cnf</strong>\r\n<span
  style=\"font-size: 13px; line-height: 19px;\">  </span>\r\n如一个定义两个实例的参考配置：\r\n<pre>[mysqld_multi]\r\nmysqld
  = /usr/local/mysql/bin/mysqld_safe\r\nmysqladmin = /usr/local/mysql/bin/mysqladmin\r\nuser
  = your_user\r\npassword = your_password\r\n\r\n[mysqld1]\r\ndatadir = /data/db/my1\r\n\r\n#连接\r\nport
  = 3306\r\nsocket = /tmp/mysql3306.sock\r\n\r\n#binlog\r\nlog-bin=/data/db/mylog1/mysql-bin\r\nbinlog_format=mixed\r\nbinlog_cache_size
  = 32M\r\nexpire_logs_days = 30\r\n\r\n[mysqld2]\r\ndatadir = /data/db/my2\r\n\r\n#连接\r\nport
  = 3307\r\nsocket = /tmp/mysql3307.sock\r\n\r\n#binlog\r\nlog-bin=/data/db/mylog2/mysql-bin\r\nbinlog_format=mixed\r\nbinlog_cache_size
  = 32M\r\nexpire_logs_days = 3</pre>\r\n"
wordpress_id: 410
wordpress_url: http://www.dcshi.com/?p=410
date: '2013-11-10 05:05:30 +0000'
date_gmt: '2013-11-10 05:05:30 +0000'
categories:
- mysql
tags: []
---
<p>在实际的开发过程中，可能会需要在一台服务器上部署多个MYSQL实例，那建议使用MYSQL官方的解决方案 mysqld_multi</p>
<p>&nbsp;</p>
<p><strong>1.修改my.cnf</strong><br />
<span style="font-size: 13px; line-height: 19px;">  </span><br />
如一个定义两个实例的参考配置：</p>
<pre>[mysqld_multi]
mysqld = /usr/local/mysql/bin/mysqld_safe
mysqladmin = /usr/local/mysql/bin/mysqladmin
user = your_user
password = your_password

[mysqld1]
datadir = /data/db/my1

#连接
port = 3306
socket = /tmp/mysql3306.sock

#binlog
log-bin=/data/db/mylog1/mysql-bin
binlog_format=mixed
binlog_cache_size = 32M
expire_logs_days = 30

[mysqld2]
datadir = /data/db/my2

#连接
port = 3307
socket = /tmp/mysql3307.sock

#binlog
log-bin=/data/db/mylog2/mysql-bin
binlog_format=mixed
binlog_cache_size = 32M
expire_logs_days = 3</pre>
<p><a id="more"></a><a id="more-410"></a><br />
&nbsp;<br />
<strong>2.创建数据目录</strong></p>
<pre>mkdir -p /data/db/my21
mkdir -p /data/db/my2
chown mysql.mysql /data/db/my1 -R
chown mysql.mysql /data/db/my2 -R</pre>
<p>&nbsp;<br />
<strong>3.初始化DB</strong></p>
<pre>/usr/local/mysql/scripts/mysql_install_db --datadir=/data/db/my1/ -uroot (mysql_install_db也是MYSQL官方自带工具)
/usr/local/mysql/scripts/mysql_install_db --datadir=/data/db/my2/ -uroot
chown mysql.mysql /data/db/my1/ -R
chown mysql.mysql /data/db/my2/ -R</pre>
<p>&nbsp;</p>
<p><strong>4. 安装工具</strong></p>
<pre>cp /usr/local/mysql/bin/my_print_defaults /usr/bin/
cp /usr/local/mysql/bin/mysqld_multi /usr/bin/</pre>
<p>&nbsp;</p>
<p><strong>5.创建、授权用户</strong></p>
<pre>CREATE USER "your_user"@"192.168.1.%" IDENTIFIED BY 'your_password';
GRANT ALL PRIVILEGES ON *.* TO "your_user"@"192.168.1.%";
flush privileges;</pre>
<p>&nbsp;</p>
<p>至此，mysql多实例配置已经完毕。我们看到多个不同的MYSQL实例是共用my.cnf的。多实例命令行管理：<br />
<strong>1.mysql启动</strong></p>
<pre>mysqld_multi start 1 启动实例1
mysqld_multi start 1-2 启动实例1,2</pre>
<p>&nbsp;</p>
<p><strong>2.mysql重启</strong></p>
<pre>mysqld_multi restart 1 重启实例1
mysqld_multi restart 1-2 重启实例1,2</pre>
<p>&nbsp;</p>
<p><strong>3.mysql关闭</strong></p>
<pre>mysqld_multi stop 1 关闭实例1
mysqld_multi stop 1-2 关闭实例1,2</pre>
<p>&nbsp;</p>
<p><strong>4.命令行登陆实例2</strong></p>
<pre>mysql -u your_user -p your_password -P3307 -S /tmp/mysql3307.sock</pre>
