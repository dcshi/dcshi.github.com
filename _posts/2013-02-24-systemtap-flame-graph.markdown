---
layout: post
status: publish
published: true
title: Systemtap生成Flame Graph(火焰图)
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
excerpt: "<p style=\"text-indent: 1em;\">通过性能分析来确定什么原因导致CPU繁忙是日常工作中长做的事情，这往往会涉及到栈性能分析。通过定期采样的方式来确定哪些代码是导致CPU繁忙的原因是一个比较粗糙的方法。一个更好的方式是创建一个定时中断来收集程序运行时的计数，函数地址，甚至整个堆栈回溯，最后打印为我们可阅读的报告.</p>\r\n&nbsp;\r\n<p
  style=\"text-indent: 1em;\">我们常用的性能分析工具有<a href=\"http://www.ibm.com/developerworks/cn/linux/l-oprof/index.html\"
  target=\"_blank\">oprofile</a>， <a href=\"http://www.ibm.com/developerworks/cn/linux/l-gnuprof.html\"
  target=\"_blank\">gprof</a>,<a href=\"http://www.ibm.com/developerworks/cn/aix/library/au-dtraceprobes.html\"
  target=\"_blank\">dtrace</a>，<a href=\"http://www.ibm.com/developerworks/cn/linux/l-systemtap/\"
  target=\"_blank\">systemtap</a> 等</p>\r\n&nbsp;\r\n\r\n<span style=\"font-size:
  24px; font-weight: bold;\">Flame Graph</span>\r\n<p style=\"text-indent: 1em;\">火焰图，是一个把采样所得到的堆栈跟踪可视化展示的工具。它是基于上面提到的性能分析工具的结果，Flame
  graph本身并不具备性能检测的能力。</p>\r\n\r\n\r\n&nbsp;\r\n\r\n下面是我为<a href=\"https://github.com/dcshi/ngx_http_qrcode_module\"
  target=\"_blank\">ngx_http_qrcode_module</a>制作的一个火焰图：\r\n<a href=\"http://flame.dcshi.com/ngx_qrcode.svg\"
  target=\"_blank\"><img src=\"http://flame.dcshi.com/ngx_qrcode.svg\" alt=\"flame
  graph for ngx_http_qrcode_module\" /></a>\r\n<ul>\r\n\t<li>每个框代表一个栈里的一个函数</li>\r\n\t<li>Y轴代表栈深度（栈桢数）。最顶端的框显示正在运行的函数，这之下的框都是调用者。在下面的函数是上面函数的父函数</li>\r\n\t<li>X轴代表采样总量。从左到右并不代表时间变化，从左到右也不具备顺序性</li>\r\n\t<li>框的宽度代表占用CPU总时间。宽的框代表的函数可能比窄的运行慢，或者被调用了更多次数。框的颜色深浅也没有任何意义</li>\r\n\t<li>如果是多线程同时采样，采样总数会超过总时间</li>\r\n</ul>\r\n&nbsp;\r\n"
wordpress_id: 287
wordpress_url: http://www.dcshi.com/?p=287
date: '2013-02-24 06:20:09 +0000'
date_gmt: '2013-02-24 06:20:09 +0000'
categories:
- "压力测试"
- nginx
tags: []
---
<p style="text-indent: 1em;">通过性能分析来确定什么原因导致CPU繁忙是日常工作中长做的事情，这往往会涉及到栈性能分析。通过定期采样的方式来确定哪些代码是导致CPU繁忙的原因是一个比较粗糙的方法。一个更好的方式是创建一个定时中断来收集程序运行时的计数，函数地址，甚至整个堆栈回溯，最后打印为我们可阅读的报告.</p>
<p>&nbsp;</p>
<p style="text-indent: 1em;">我们常用的性能分析工具有<a href="http://www.ibm.com/developerworks/cn/linux/l-oprof/index.html" target="_blank">oprofile</a>， <a href="http://www.ibm.com/developerworks/cn/linux/l-gnuprof.html" target="_blank">gprof</a>,<a href="http://www.ibm.com/developerworks/cn/aix/library/au-dtraceprobes.html" target="_blank">dtrace</a>，<a href="http://www.ibm.com/developerworks/cn/linux/l-systemtap/" target="_blank">systemtap</a> 等</p>
<p>&nbsp;</p>
<p><span style="font-size: 24px; font-weight: bold;">Flame Graph</span></p>
<p style="text-indent: 1em;">火焰图，是一个把采样所得到的堆栈跟踪可视化展示的工具。它是基于上面提到的性能分析工具的结果，Flame graph本身并不具备性能检测的能力。</p>
<p>&nbsp;</p>
<p>下面是我为<a href="https://github.com/dcshi/ngx_http_qrcode_module" target="_blank">ngx_http_qrcode_module</a>制作的一个火焰图：<br />
<a href="http://flame.dcshi.com/ngx_qrcode.svg" target="_blank"><img src="http://flame.dcshi.com/ngx_qrcode.svg" alt="flame graph for ngx_http_qrcode_module" /></a></p>
<ul>
<li>每个框代表一个栈里的一个函数</li>
<li>Y轴代表栈深度（栈桢数）。最顶端的框显示正在运行的函数，这之下的框都是调用者。在下面的函数是上面函数的父函数</li>
<li>X轴代表采样总量。从左到右并不代表时间变化，从左到右也不具备顺序性</li>
<li>框的宽度代表占用CPU总时间。宽的框代表的函数可能比窄的运行慢，或者被调用了更多次数。框的颜色深浅也没有任何意义</li>
<li>如果是多线程同时采样，采样总数会超过总时间</li>
</ul>
<p>&nbsp;<br />
<a id="more"></a><a id="more-287"></a><br />
<span style="font-size: 24px; font-weight: bold;">Systemtap</span></p>
<p style="text-indent: 1em;">SystemTap 是监控和跟踪运行中的 Linux 内核的操作的动态方法。这句话的关键词是动态，因为 SystemTap 没有使用工具构建一个特殊的内核，而是允许您在运行时动态地安装该工具。它通过一个名为Kprobes 的应用编程接口（API）来实现该目的。</p>
<p style="text-indent: 1em;">systemtap的安装与使用请参考<a href="http://sourceware.org/systemtap/wiki" target="_blank">http://sourceware.org/systemtap/wiki</a></p>
<p>&nbsp;</p>
<p>生成上面火焰图的systemtap脚本qrcode.stap如下:</p>
<pre>    global bt;
    global quit = 0

    probe timer.profile {
        if (pid() == target()) {
            if (!quit) {
                bt[backtrace(), ubacktrace()] &lt;&lt;&lt; 1
            } else {
                foreach ([sys, usr] in bt- limit 1000) {
                    print_stack(sys)
                    print_ustack(usr)
                    printf("\t%d\n", @count(bt[sys, usr]))
                }
                exit()
            }
        }
    }

    probe timer.s(20) {
        quit = 1
    }</pre>
<p>&nbsp;</p>
<p>运行这个脚本的一条典型命令行是：</p>
<pre>sudo stap --ldd -d /usr/local/sbin/nginx  \

            --all-modules -D MAXMAPENTRIES=1024\

            -D MAXACTION=20000 \

            -D MAXTRACE=100 \

            -D MAXSTRINGLEN=4096 \

            -D MAXBACKTRACE=100 -x11111 fire.stp > a.out</pre>
<p>&nbsp;</p>
<p style="text-indent: 1em">
假设这里测量的 nginx worker 进程的 pid 是11111. 大约过 20 多秒之后，该命令会退出并生成 a.out。其中 -D 选项定义的那些宏的值是为了放宽 systemtap 内部的很多限制。例如，MAXTRACE宏用于控制调用栈的深度，默认只有 20. 所以当实际的调用栈深度超过 20时（事实上，经常会超时），但会发生调用栈信息的自动截断，从而影响火焰图的绘制。</p>
<pre>MAXNESTING - The maximum number of recursive function call levels. The default is 10.

MAXSTRINGLEN - The maximum length of strings. The default is 256 bytes for 32 bit machines and 512 bytes for all other machines.

MAXTRYLOCK - The maximum number of iterations to wait for locks on global variables before declaring possible deadlock and skipping the probe. The default is 1000.

MAXACTION - The maximum number of statements to execute during any single probe hit. The default is 1000.

MAXMAPENTRIES - The maximum number of rows in an array if the array size is not specified explicitly when declared. The default is 2048.

MAXERRORS - The maximum number of soft errors before an exit is triggered. The default is 0.

MAXSKIPPED - The maximum number of skipped reentrant probes before an exit is triggered. The default is 100.

MINSTACKSPACE - The minimum number of free kernel stack bytes required in order to run a probe handler. This number should be large enough for the probe handler's own needs, plus a safety margin. The default is 1024.</pre>
<p>&nbsp;</p>
<p>输出文件，然后从此文件生成火焰图文件 a.svg：</p>
<pre>    perl stackcollapse-stap.pl a.out > a.out2

    perl flamegraph.pl a.out2 > a.svg</pre>
<p>这里使用的两个 perl 脚本（.pl 文件）来自 Brendan Gregg 放在 GitHub 上的 flamegraph 项目：</p>
<p><a href="https://github.com/brendangregg/FlameGraph" target="_blank">https://github.com/brendangregg/FlameGraph</a></p>
<p>&nbsp;</p>
<p style="text-indent: 1em">值得提醒的是，在运行 systemtap 脚本期间，被采样的 nginx worker 进程（或者其他任意进程）须是足够繁忙的，否则systemtap 脚本可能会长时间不退出。一般我在上面第一步运行 stap 之前会先用 ab 工具对 nginx服务器保持压力，至少得持续到 stap 命令自动退出以后。</p>
<p>&nbsp;</p>
<p>参考:<br />
<a href="https://groups.google.com/forum/?fromgroups=#!starred/openresty/u-puKWWONMk" target="_blank">https://groups.google.com/forum/?fromgroups=#!starred/openresty/u-puKWWONMk</a><br />
<a href="http://dtrace.org/blogs/brendan/2011/12/16/flame-graphs/" target="_blank">http://dtrace.org/blogs/brendan/2011/12/16/flame-graphs/</a><br />
<a href="http://dtrace.org/blogs/brendan/2012/03/17/linux-kernel-performance-flame-graphs/" target="_blank">http://dtrace.org/blogs/brendan/2012/03/17/linux-kernel-performance-flame-graphs/</a></p>
