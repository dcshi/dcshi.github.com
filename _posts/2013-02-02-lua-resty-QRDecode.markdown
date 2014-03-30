---
layout: post
status: publish
published: true
title: lua-resty-QRDecode - 二维码解码模块
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
excerpt: "目前我们常见的二维码解码场景都是通过手机的摄像头扫描，并结合相应的SDK解码。\r\n但如果你有跟我们类似的需求：在linux后台解析二维码图片，得到二维码对应的明文内容。
  则可参考本文。\r\n\r\n&nbsp;\r\n\r\n为什么是lua-resty-QRDecode?\r\n<p style=\"text-indent:
  2em\">这只是对外提供接口的一个形式(可在ngx_lua直接调用），你可以完全基于你的需求，写一个php的扩展，或者c/c++的cgi/fastcgi调用。这不影响我们下面的讨论</p>\r\n\r\n&nbsp;\r\n\r\nshow我的系统信息:\r\n<pre>uname
  -a\r\nLinux Aliyun64 3.2.0-29-generic #46-Ubuntu SMP Fri Jul 27 17:03:23 UTC 2012
  x86_\r\n\r\ngcc -v\r\ngcc version 4.6.3\r\n\r\npython -V\r\nPython 2.7.3\r\n</pre>\r\n&nbsp;\r\n\r\n一.
  <strong>安装&amp;编译 zxing </strong>（Multi-format 1D/2D barcode image processing library）\r\n下载zxing
  当前最新版本是2.1(更多参考 <a href=\"http://code.google.com/p/zxing/\" target=\"_blank\">http://code.google.com/p/zxing/</a>)
  <a href=\"http://zxing.googlecode.com/files/ZXing-2.1.zip\" target=\"_blank\">http://zxing.googlecode.com/files/ZXing-2.1.zip</a>
  。 zxing的zip包里多个语言版本的dev包(c++/java/c#等)，cpp是今天的主角，安装步骤如下(请参考cpp/README)。\r\n<ol>\r\n\t<li>python
  scons/scons.py lib #编译生成libzxing.a</li>\r\n\t<li>python scons/scons.py tests #编译单元测试模块，编译前需要确保你已经安装libcppunit-dev</li>\r\n\t<li>python
  scons/scons.py zxing #编译生成一个命令行测试工具zxing，编译前需要确保你已经安装libmagick++-dev。</li>\r\n</ol>\r\n<p
  style=\"text-indent: 2em\">以上编译生成的目标文件都在cpp/build下面，分别对应三个文件libzxing.a testrunner
  zxing 。 如果你的编译跟我一样顺利的话，你可以用zxing这个工具来进行二维码解码测试了。google也提供了一些二维码图片测试素材:http://code.google.com/p/zxing/downloads/detail?name=ZXing-2.1-testdata.zip).</p>\r\n"
wordpress_id: 242
wordpress_url: http://www.dcshi.com/?p=242
date: '2013-02-02 10:56:30 +0000'
date_gmt: '2013-02-02 10:56:30 +0000'
categories:
- nginx
- ngx_lua
tags: []
---
<p>目前我们常见的二维码解码场景都是通过手机的摄像头扫描，并结合相应的SDK解码。<br />
但如果你有跟我们类似的需求：在linux后台解析二维码图片，得到二维码对应的明文内容。 则可参考本文。</p>
<p>&nbsp;</p>
<p>为什么是lua-resty-QRDecode?</p>
<p style="text-indent: 2em">这只是对外提供接口的一个形式(可在ngx_lua直接调用），你可以完全基于你的需求，写一个php的扩展，或者c/c++的cgi/fastcgi调用。这不影响我们下面的讨论</p>
<p>&nbsp;</p>
<p>show我的系统信息:</p>
<pre>uname -a
Linux Aliyun64 3.2.0-29-generic #46-Ubuntu SMP Fri Jul 27 17:03:23 UTC 2012 x86_

gcc -v
gcc version 4.6.3

python -V
Python 2.7.3
</pre>
<p>&nbsp;</p>
<p>一. <strong>安装&amp;编译 zxing </strong>（Multi-format 1D/2D barcode image processing library）<br />
下载zxing 当前最新版本是2.1(更多参考 <a href="http://code.google.com/p/zxing/" target="_blank">http://code.google.com/p/zxing/</a>) <a href="http://zxing.googlecode.com/files/ZXing-2.1.zip" target="_blank">http://zxing.googlecode.com/files/ZXing-2.1.zip</a> 。 zxing的zip包里多个语言版本的dev包(c++/java/c#等)，cpp是今天的主角，安装步骤如下(请参考cpp/README)。</p>
<ol>
<li>python scons/scons.py lib #编译生成libzxing.a</li>
<li>python scons/scons.py tests #编译单元测试模块，编译前需要确保你已经安装libcppunit-dev</li>
<li>python scons/scons.py zxing #编译生成一个命令行测试工具zxing，编译前需要确保你已经安装libmagick++-dev。</li>
</ol>
<p style="text-indent: 2em">以上编译生成的目标文件都在cpp/build下面，分别对应三个文件libzxing.a testrunner zxing 。 如果你的编译跟我一样顺利的话，你可以用zxing这个工具来进行二维码解码测试了。google也提供了一些二维码图片测试素材:http://code.google.com/p/zxing/downloads/detail?name=ZXing-2.1-testdata.zip).</p>
<p><a id="more"></a><a id="more-242"></a><br />
&nbsp;</p>
<p style="text-indent: 2em">SConscript类似我们的makefile，从该文件可以看到，zxing的源码程序在cpp/magick/src/main.cpp ,其包含的lib和头文件都在SConscript:60里找到定义。编译libzxing.a静态库前，我修改了下SConscript:9 ,总使用PIC选项来编译。这样做的目的是后续编译我们的自定义的so后，so就可以独立libzxing.a而运行(请参考 http://hi.baidu.com/proinsight/item/150dd3e6e506d1a9c10d7552)</p>
<p>&nbsp;</p>
<p style="text-indent: 2em">我没有找到cpp 的API说明文档，但没有关系。看main.cpp的源码，可以比较好理解怎么使用zxing的API 。以便我们可以基于此来封装自己的qrdecode模块。（正如前面所说，怎么封装是业务相关。如果是c/c++的cgi，可以直接基于原始API来开发；如果是lua/php则需要编译成so）</p>
<p>&nbsp;</p>
<p>在main.cpp里，核心的函数是test_image_hybrid, test_image_global .分别对应的是两种不一样的decode算法。</p>
<p>&nbsp;</p>
<p>二. <strong>编译自己的libqrdecode.so</strong><br />
这里列出下面讨论的前提:</p>
<ol>
<li>编译libzxing.a时总启用PIC编译选项</li>
<li>libmagick++-dev已经安装</li>
</ol>
<p>源码文件在: xxxxx<br />
请根据自身信息修改makefile里的INC, LIB<br />
INC := -I/usr/include/ImageMagick/ -I../../core/src/<br />
LIB := -lMagickWand -lMagick++ -lMagickCore -L../../build/ -lzxing</p>
<p>&nbsp;</p>
<p>顺利的话，我们需要的libqrdecode.so就生成了。</p>
<p>&nbsp;</p>
<p><strong>三. 如何使用</strong><br />
具体的使用方式请参考: <a href="https://github.com/dcshi/lua-resty-QRDecode/blob/master/README.md" title="lua-resty-QRDecode" target="_blank">https://github.com/dcshi/lua-resty-QRDecode/blob/master/README.md</a></p>
<p>&nbsp;</p>
<p><strong>安装过程中遇到的问题：</strong><br />
我分别在两台ubuntu vps上通过apt-get安装，当然安装都很顺利，但是在其中一台vps上编译zxing时候遇到下面的错误：<br />
/usr/lib/gcc/i686-linux-gnu/4.6/../../../i386-linux-gnu/libMagickCore.so: undefined reference to `gztell64@ZLIB_1.2.3.3'<br />
/usr/lib/gcc/i686-linux-gnu/4.6/../../../i386-linux-gnu/libMagickCore.so: undefined reference to `gzseek64@ZLIB_1.2.3.3'<br />
/usr/lib/gcc/i686-linux-gnu/4.6/../../../i386-linux-gnu/libMagickCore.so: undefined reference to `gzopen64@ZLIB_1.2.3.3'</p>
<p>&nbsp;</p>
<p style="text-indent: 2em">我初步是怀疑libMagickCore.so有问题，因为两个vps都是通过apt-get进行安装的，所以我比对了/ect/apt/source.list. 发现配的源是不一样的，于是我就把正确的机器源的配置同步到出现问题的这个vps来，重新执行apt-get update，并不之前安装的libmagick++-dev remove 再重新安装。</p>
<p>&nbsp;</p>
<p>show下source.list ：<br />
deb http://cn.archive.ubuntu.com/ubuntu/ precise main restricted universe multiverse<br />
deb http://cn.archive.ubuntu.com/ubuntu/ precise-security main restricted universe multiverse<br />
deb http://cn.archive.ubuntu.com/ubuntu/ precise-updates main restricted universe multiverse<br />
deb http://cn.archive.ubuntu.com/ubuntu/ precise-proposed main restricted universe multiverse<br />
deb http://cn.archive.ubuntu.com/ubuntu/ precise-backports main restricted universe multiverse<br />
deb-src http://cn.archive.ubuntu.com/ubuntu/ precise main restricted universe multiverse<br />
deb-src http://cn.archive.ubuntu.com/ubuntu/ precise-security main restricted universe multiverse<br />
deb-src http://cn.archive.ubuntu.com/ubuntu/ precise-updates main restricted universe multiverse<br />
deb-src http://cn.archive.ubuntu.com/ubuntu/ precise-proposed main restricted universe multiverse<br />
deb-src http://cn.archive.ubuntu.com/ubuntu/ precise-backports main restricted universe multiverse</p>
