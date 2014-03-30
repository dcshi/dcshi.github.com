---
layout: post
status: publish
published: true
title: ubuntu12.04 安装systemtap
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 124
wordpress_url: http://www.dcshi.com/?p=124
date: '2012-10-17 13:54:33 +0000'
date_gmt: '2012-10-17 13:54:33 +0000'
categories:
- systemtap
tags:
- systemtap
---
<p>首先通过如下命令查看系统内核版本</p>
<pre>root@ubuntu:/usr/local# cat /etc/issue
Ubuntu 12.04 LTS \n \l
root@ubuntu:/usr/local# uname -r
3.2.0-23-generic-pae</pre>
<p>&nbsp;</p>
<p>systemtap安装有以下两种途径：<br />
1)apt-get 安装：sudo apt-get install systematic<br />
2)源码安装: ./configure --prefix=/opt/systemtap --disable-docs --disable-publican --disable-refdocs &amp;&amp; make -j8;sudo make install<br />
stap可执行文件默认安装在/opt/systemtap/bin/stap,可以通过建立软连接 sudo ln -s /opt/systemtap/bin/stap /usr/sbin/stap<br />
apt-get的systemtap版本一般不是最新，所以更建议通过源码安装,源码可以在这里下载:<a title="下载" href="http://sourceware.org/systemtap/ftp/releases/" target="_blank">http://sourceware.org/systemtap/ftp/releases/</a></p>
<p>&nbsp;</p>
<p>安装完systemtap以后执行:stap -e 'probe kernel.function("sys_open") {log("hello world") exit()}'<br />
预期的结果是输出hello world.但一般会得到如下报错：</p>
<pre>Checking "/lib/modules/3.2.0-23-generic-pae/build/.config" failed with error: No such file or directory</pre>
<p>为了解决这个error，可依次执行如下操作</p>
<pre>$ cd $HOME
$ sudo apt-get install dpkg-dev debhelper gawk
$ mkdir tmp
$ cd tmp
$ sudo apt-get build-dep --no-install-recommends linux-image-$(uname -r)
$ apt-get source linux-image-$(uname -r)
$ cd linux-3.2.0 (这里是我的内核版本，请根据你实际版本号操作)
$ fakeroot debian/rules clean
$ AUTOBUILD=1 fakeroot debian/rules binary-generic skipdbg=false    （这里重新编译一个可调试的内核版本，所以会比较费时）
$ sudo dpkg -i ../linux-image-3.2.0-32-generic-dbgsym_3.2.0-32.51_i386.ddeb</pre>
<p>&nbsp;</p>
<p>上面倒数第二步会产生如下三个文件：</p>
<pre>linux-image-3.2.0-32-generic_3.2.0-32.51_i386.deb
linux-headers-3.2.0-32-generic_3.2.0-32.51_i386.deb
linux-image-3.2.0-32-generic-dbgsym_3.2.0-32.51_i386.ddeb</pre>
<p>通过uname -r确认内核版本与生成的kernel包的版本是否一致（我本来的内核版本是3.2.0-23-generic-pae，生成出来的kernel包是3.2.0-32-generic，不带pae，即版本是不一致的），对于不一致的情况，需要重新安装kernel：<br />
sudo dpkg -i linux-image-3.2.0-32-generic_3.2.0-32.51_i386.deb<br />
安装完以后需要重启linux（sudo shutdown -r now），然后选择新安装的内核进入系统，再安装dbgsym包(即上述提到的最后一步)：</p>
<pre>sudo dpkg -i your-path/linux-image-3.2.0-32-generic-dbgsym_3.2.0-32.51_i386.ddeb</pre>
<p>&nbsp;</p>
<p>到这里，你可能迫不及待要想去执行stap -e 'probe kernel.function("sys_open") {log("hello world") exit()}'。<br />
你可能得到的结果是<br />
Checking " /lib/modules/3.2.0-32-generic/build/Makefile " failed with error: No such file or directory<br />
Ensure kernel development headers &amp; makefiles are installed.<br />
这个比较好解决，重新安装下kernel header好了：sudo apt-get install linux-headers-`uname -r`</p>
<p>&nbsp;</p>
<p>至此内核追踪已经可以执行，但module的信息还需要多做些工作，执行以下命令：</p>
<pre>sudo apt-get install elfutils
for file in `find /usr/lib/debug -name '*.ko' -print`
do
      buildid=`eu-readelf -n $file| grep Build.ID: | awk '{print $3}'`
      dir=`echo $buildid | cut -c1-2`
      fn=`echo $buildid | cut -c3-`
      rm -fr /usr/lib/debug/.build-id
      mkdir -p /usr/lib/debug/.build-id/$dir
      ln -s $file /usr/lib/debug/.build-id/$dir/$fn
      ln -s $file /usr/lib/debug/.build-id/$dir/${fn}.debug
done</pre>
<p>&nbsp;</p>
<p>到此为止，希望你的systemtap能成功run起来:sudo stap -e 'probe kernel.function("sys_open") {log("hello world") exit()}'<br />
提醒下：第一次之下systemtap可能会要等1min左右的时间才会输出hello world, 年轻人给点耐心</p>
<p>&nbsp;</p>
<p>参考:<br />
<a href="http://sourceware.org/systemtap/wiki/SystemtapOnUbuntu" target="_blank">http://sourceware.org/systemtap/wiki/SystemtapOnUbuntu</a></p>
<div><a href="http://www.ningoo.net/html/2010/use_systemtap_on_ubuntu.html">http://www.ningoo.net/html/2010/use_systemtap_on_ubuntu.html</a></div>
<div><a href="http://www.cnblogs.com/hdflzh/archive/2012/07/25/2608910.html">http://www.cnblogs.com/hdflzh/archive/2012/07/25/2608910.html</a></div>
<div><a href="http://www.ibm.com/developerworks/cn/linux/l-systemtap/">http://www.ibm.com/developerworks/cn/linux/l-systemtap/</a></div>
