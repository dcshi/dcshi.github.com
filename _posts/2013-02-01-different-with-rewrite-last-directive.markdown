---
layout: post
status: publish
published: true
title: nginx rewrite 指令last break区别
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
excerpt: "nginx 的官方注释是这样的：\r\n<pre>last\r\n   stops processing the current set of
  ngx_http_rewrite_module directives followed by a search for a new location matching
  the changed URI;\r\n\r\nbreak \r\n   stops processing the current set of ngx_http_rewrite_module
  directives;</pre>\r\n&nbsp;\r\n\r\n我们知道nginx运行分十一个执行阶段，上面说提到的ngx_http_rewrite_mode,可以理解为其中一个阶段-rewrite阶段。​\r\n<pre>typedef
  enum {\r\n    NGX_HTTP_POST_READ_PHASE = 0,\r\n    NGX_HTTP_SERVER_REWRITE_PHASE,\r\n
  \   NGX_HTTP_FIND_CONFIG_PHASE,\r\n    NGX_HTTP_REWRITE_PHASE,           //rewrite阶段在这里\r\n
  \   NGX_HTTP_POST_REWRITE_PHASE,\r\n    NGX_HTTP_PREACCESS_PHASE,\r\n    NGX_HTTP_ACCESS_PHASE,\r\n
  \   NGX_HTTP_POST_ACCESS_PHASE,\r\n    NGX_HTTP_TRY_FILES_PHASE,\r\n    NGX_HTTP_CONTENT_PHASE,\r\n
  \   NGX_HTTP_LOG_PHASE\r\n} ngx_http_phases;</pre>\r\n&nbsp;\r\n\r\n所以我们再来理解last与break的区别：\r\nlast：
  停止当前这个请求，并根据rewrite匹配的规则重新发起一个请求。新请求又从第一阶段开始执行...\r\nbreak：相对last，break并不会重新发起一个请求，只是跳过当前的rewrite阶段，并执行本请求后续的执行阶段...\r\n\r\n&nbsp;\r\n\r\n我们来看一个例子：\r\n<pre>server
  {\r\n    listen 80 default_server;\r\n    server_name dcshi.com;\r\n    root www;\r\n\r\n
  \   location /break/ {\r\n        rewrite ^/break/(.*) /test/$1 break;\r\n        echo
  \"break page\";\r\n    } \r\n\r\n    location /last/ {\r\n         rewrite ^/last/(.*)
  /test/$1 last;\r\n         echo \"last page\";\r\n    }    \r\n\r\n    location
  /test/ {\r\n       echo \"test page\";\r\n    }\r\n}</pre>\r\n&nbsp;\r\n"
wordpress_id: 172
wordpress_url: http://www.dcshi.com/?p=172
date: '2013-02-01 17:09:50 +0000'
date_gmt: '2013-02-01 17:09:50 +0000'
categories:
- nginx
tags: []
---
<p>nginx 的官方注释是这样的：</p>
<pre>last
   stops processing the current set of ngx_http_rewrite_module directives followed by a search for a new location matching the changed URI;

break 
   stops processing the current set of ngx_http_rewrite_module directives;</pre>
<p>&nbsp;</p>
<p>我们知道nginx运行分十一个执行阶段，上面说提到的ngx_http_rewrite_mode,可以理解为其中一个阶段-rewrite阶段。​</p>
<pre>typedef enum {
    NGX_HTTP_POST_READ_PHASE = 0,
    NGX_HTTP_SERVER_REWRITE_PHASE,
    NGX_HTTP_FIND_CONFIG_PHASE,
    NGX_HTTP_REWRITE_PHASE,           //rewrite阶段在这里
    NGX_HTTP_POST_REWRITE_PHASE,
    NGX_HTTP_PREACCESS_PHASE,
    NGX_HTTP_ACCESS_PHASE,
    NGX_HTTP_POST_ACCESS_PHASE,
    NGX_HTTP_TRY_FILES_PHASE,
    NGX_HTTP_CONTENT_PHASE,
    NGX_HTTP_LOG_PHASE
} ngx_http_phases;</pre>
<p>&nbsp;</p>
<p>所以我们再来理解last与break的区别：<br />
last： 停止当前这个请求，并根据rewrite匹配的规则重新发起一个请求。新请求又从第一阶段开始执行...<br />
break：相对last，break并不会重新发起一个请求，只是跳过当前的rewrite阶段，并执行本请求后续的执行阶段...</p>
<p>&nbsp;</p>
<p>我们来看一个例子：</p>
<pre>server {
    listen 80 default_server;
    server_name dcshi.com;
    root www;

    location /break/ {
        rewrite ^/break/(.*) /test/$1 break;
        echo "break page";
    } 

    location /last/ {
         rewrite ^/last/(.*) /test/$1 last;
         echo "last page";
    }    

    location /test/ {
       echo "test page";
    }
}</pre>
<p>&nbsp;<br />
<a id="more"></a><a id="more-172"></a><br />
<strong>请求:</strong>http://dcshi.com/break/***<br />
<strong>输出:</strong> break page<br />
<strong>分析：</strong>正如上面讨论所说，break是跳过当前请求的rewrite阶段，并继续执行本请求的其他阶段，很明显，对于/foo 对应的content阶段的输出为 echo "break page"; (content阶段，可以简单理解为产生数据输出的阶段，如返回静态页面内容也是在content阶段；echo指令也是运行在content阶段，一般情况下content阶段只能对应一个输出指令，如同一个location配置两个echo，最终只会有一个echo指令被执行)；当然如果你把/break/里的echo 指令注释，然后再次访问/break/xx会报404，这也跟我们预期一样：虽然/break/xx被重定向到/test/xx,但是break指令不会重新开启一个新的请求继续匹配，所以nginx是不会匹配到下面的/test/这个location；在echo指令被注释的情况下，/break/ 这location里只能执行nginx默认的content指令，即尝试找/test/xx这个html页面并输出起内容，事实上，这个页面不存在，所以会报404的错误。</p>
<p>&nbsp;</p>
<p><strong>请求: </strong>http://dcshi.com/last/***<br />
<strong>输出:</strong> test page<br />
<strong>分析:</strong> last与break最大的不同是，last会重新发起一个新请求，并重新匹配location，所以对于/last,重新匹配请求以后会匹配到/test/,所以最终对应的content阶段的输出是test page;</p>
<p>&nbsp;</p>
<p>假设你对nginx的运行阶段有一个大概的理解，对理解last与break就没有问题了。</p>
