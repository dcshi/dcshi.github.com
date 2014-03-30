---
layout: post
status: publish
published: true
title: ngx_http_qrcode_module - nginx二维码模块(qrcode module)更新
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
excerpt: "<a href=\"https://github.com/dcshi/ngx_http_qrcode_module/\" target=\"_blank\">ngx_http_qrcode_module</a>
  is a an addon for Nginx to generate and serve QR code.(nginx qrcode module)\r\n\r\n&nbsp;\r\n\r\n<p
  style=\"text-indent: 2em\">本次更新是增加变量支持，变量规则与nginx的set指令一致，支持变量组合等。可允许根据每个请求来自定义二维码属性，以便生成不同的二维码图片.</p>\r\n\r\n&nbsp;\r\n\r\n先来看一个例子，nginx.conf配置如下：\r\n<pre>worker_processes
  \ 1;      \r\nerror_log  logs​/error.log  debug;\r\n\r\nevents {\r\n    worker_connections
  \ 1024;\r\n}\r\n\r\nhttp {\r\n\r\n    server {\r\n        listen       1314;\r\n
  \       server_name  localhost;\r\n\r\n        location /qr {\r\n            qrcode_fg_color
  $arg_fg_color;\r\n            qrcode_bg_color $arg_bg_color;\r\n            qrcode_level
  $arg_level;\r\n            qrcode_hint $arg_hint;\r\n            qrcode_size $arg_size;\r\n
  \           qrcode_margin $arg_margin;\r\n            qrcode_version $arg_ver;\r\n
  \           qrcode_casesensitive $arg_case;\r\n            qrcode_txt \"${arg_txt}_hello_world\";
  \ ＃变量与常量的组合\r\n\r\n            qrcode_gen;\r\n        }   \r\n    }   \r\n}</pre>\r\n<pre>curl
  \"http://localhost:1314/qr?size=6&fg_color=00FF00&bg_color=fff700&case=1&txt=12a&margin=2&level=0&hint=2&ver=2\"</pre>\r\n&nbsp;\r\n"
wordpress_id: 266
wordpress_url: http://www.dcshi.com/?p=266
date: '2013-02-04 15:45:14 +0000'
date_gmt: '2013-02-04 15:45:14 +0000'
categories:
- nginx
tags: []
---
<p><a href="https://github.com/dcshi/ngx_http_qrcode_module/" target="_blank">ngx_http_qrcode_module</a> is a an addon for Nginx to generate and serve QR code.(nginx qrcode module)</p>
<p>&nbsp;</p>
<p style="text-indent: 2em">本次更新是增加变量支持，变量规则与nginx的set指令一致，支持变量组合等。可允许根据每个请求来自定义二维码属性，以便生成不同的二维码图片.</p>
<p>&nbsp;</p>
<p>先来看一个例子，nginx.conf配置如下：</p>
<pre>worker_processes  1;      
error_log  logs​/error.log  debug;

events {
    worker_connections  1024;
}

http {

    server {
        listen       1314;
        server_name  localhost;

        location /qr {
            qrcode_fg_color $arg_fg_color;
            qrcode_bg_color $arg_bg_color;
            qrcode_level $arg_level;
            qrcode_hint $arg_hint;
            qrcode_size $arg_size;
            qrcode_margin $arg_margin;
            qrcode_version $arg_ver;
            qrcode_casesensitive $arg_case;
            qrcode_txt "${arg_txt}_hello_world";  ＃变量与常量的组合

            qrcode_gen;
        }   
    }   
}</pre>
<pre>curl "http://localhost:1314/qr?size=6&amp;fg_color=00FF00&amp;bg_color=fff700&amp;case=1&amp;txt=12a&amp;margin=2&amp;level=0&amp;hint=2&amp;ver=2"</pre>
<p>&nbsp;<br />
<a id="more"></a><a id="more-266"></a><br />
<strong>调试</strong><br />
我建议把error log日志级别设置为debug，日志文件将会打印出本次请求，所设置的二维码属性：</p>
<pre>2013/02/03 02:15:46 [debug] 976#0: *1 fg_color 0ff00, bg_color fff700, level 0, hint 2, size 6, margin 2, version 2, casesensitive 1, txt 12a</pre>
<p>&nbsp;</p>
<p><strong>错误定位</strong><br />
如果后台返回500错误，请先打开日志文件，从日志文件来定位出问题；一般来说是传入参数错误所致，如传入的color格式不对或者level取值范围不对等(请参考下面指令说明)。</p>
<p>&nbsp;</p>
<p><strong>性能</strong><br />
性能取决与二维码的属性，如生成的图片大小等；模块是基于原生的libqrencode库，而且生成-&gt;输出没有任何的磁盘io操作，性能也会比php扩展等实现要好。<br />
我的另一<a href="http://www.dcshi.com/?p=138" target="_blank">文章</a>，曾介绍搭配nginx的memcache模块(<a href="http://wiki.nginx.org/HttpSRCacheModule" target="_blank">SR Cache</a>)，不用写一句代码即可实现高效的二维码编码服务。</p>
<p>&nbsp;</p>
<p><strong>指令说明:</strong><br />
Syntax: qrcode_fg_color color<br />
Default: qrcode_fg_color 000000<br />
Context: http, server, location<br />
Description: set the color of QRcode.</p>
<p>&nbsp;</p>
<p>Syntax: qrcode_bg_color color<br />
Default: qrcode_bg_color FFFFFF<br />
Context: http, server, location<br />
Description: set the background color of QRcode.</p>
<p>&nbsp;</p>
<p>Syntax: qrcode_level level<br />
Default: qrcode_level 0<br />
Context: http, server, location<br />
Description: level of error correction, [0:3]. Refer tohttp://fukuchi.org/works/qrencode/manual/qrencode_8h.html#a35d70229ba985c61bbdab27b0f65740e</p>
<p>&nbsp;</p>
<p>Syntax: qrcode_hint hint<br />
Default: qrcode_hint 2<br />
Context: http, server, location<br />
Description: encoding mode. Refer to http://fukuchi.org/works/qrencode/manual/qrencode_8h.html#ab7ec78b96e63c8f019bb328a8d4f55db</p>
<p>&nbsp;</p>
<p>Syntax: qrcode_size size<br />
Default: qrcode_size 4<br />
Context: http, server, location<br />
Description: size of qrcode image.</p>
<p>&nbsp;</p>
<p>Syntax: qrcode_margin margin<br />
Default: qrcode_margin 4<br />
Context: http, server, location<br />
Description: margin of qrcode image.</p>
<p>&nbsp;</p>
<p>Syntax: qrcode_version version<br />
Default: qrcode_version 1<br />
Context: http, server, location<br />
Description: version of the symbol. it should less 10.</p>
<p>&nbsp;</p>
<p>Syntax: qrcode_casesensitive [0 | 1]<br />
Default: qrcode_casesensitive off<br />
Context: http, server, location<br />
Description: case-sensitive(1) or not(0)</p>
<p>&nbsp;</p>
<p>Syntax: qrcode_txt txt<br />
Default: none<br />
Context: http, server, location<br />
Description: the txt you want to encode. </p>
<p>&nbsp;</p>
<p>Syntax: qrcode_gen<br />
Default: none<br />
Context: http, server, location<br />
Description: generate QRcode.</p>
