---
layout: post
status: publish
published: true
title: webscoket+nodejs+redis实现instagr real-time API
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 23
wordpress_url: http://www.dcshi.com/?p=23
date: '2011-10-15 10:30:20 +0000'
date_gmt: '2011-10-15 10:30:20 +0000'
categories:
- redis
- nodejs
tags:
- redis
- nodejs
---
<p align="left">如何实现数据实时推送：</p>
<p align="left"><a href="https://github.com/asalant/Realtime-Demo" target="_blank">https://github.com/asalant/Realtime-Demo</a></p>
<p align="left">Our real-time API serves a couple of basic needs: First, instead of polling the Instagram servers to check to see if there are new photos available, you can rely upon our servers to POST to a callback URL on your server when new data is available. Second, developers running servers like <a href="http://www.tornadoweb.org/" target="_blank">Tornado</a> or <a href="http://nodejs.org/" target="_blank">Node.js</a> can provide real-time experiences of events to their users.</p>
<p align="left">简介：</p>
<p align="left">作者用了redis的pub/sub机制来实现实时推送功能</p>
<p align="left">首先你必须要了解instagr的REAK-TIME API的工作原理：<a href="http://instagr.am/developer/realtime/" target="_blank">http://instagr.am/developer/realtime/</a></p>
<p align="left">假设你看完了上面的基本介绍，那我们开始吧：</p>
<p align="left">如果instagr有新的数据，他会主动把数据POST到你的callback API，</p>
<p align="left">此时你就需要用到redis的pub/sub机制了，把获取到的POST数据，通过调用publish把数据发布，</p>
<p align="left">server.js监测到message事件，说明有新的订阅数据到达，server会把数据发送到订阅了当前challenge的client，同时把数据保存到一个list中（为什么要保存？这保证了你server.js重新启动以后，还可以取到最新的一些数据，如果是机器重启呢？）</p>
<p align="left">so easy？let’s begin….</p>
<p align="left">一.从这里获取Real-time Demo 源码 <a href="https://github.com/asalant/Realtime-Demo" target="_blank">https://github.com/asalant/Realtime-Demo</a>.</p>
<p align="left">二.（如果你还没有安装nodejs，请先安装 <a href="https://github.com/joyent/node/wiki/Installation" target="_blank">https://github.com/joyent/node/wiki/Installation</a>）</p>
<p align="left">需要安装一些依赖的软件包：redis，socket.io,express,jade,这些都是通过npm来安装即可（curl http://npmjs.org/install.sh | sh）</p>
<p align="left">如redis的安装：在realtime-demo的根目录下面执行：</p>
<p align="left">sudo npm install redis #执行成功以后会提示redis@0.6.0 ./node_modules/redis这样的信息，实质上就是在当前目录下的node_modules目录下添加一个redis模块</p>
<p align="left">（这里redis不是指redis程序，是指redis的nodejs客户端，The project and repo are called “node_redis”, but in npm it is called “redis”）</p>
<p>三.安装localtunnel(<a href="https://github.com/progrium/localtunnel#readme" target="_blank">https://github.com/progrium/localtunnel#readme</a>)</p>
<p align="left">介绍：instant public tunnel to your local web server，In order to run the application locally, you need a way for the Instagram API to publish updates to the app. It does this through webhook callbacks so your app needs to be available on the public internet.localtunnel is a good solution for this.</p>
<p align="left">$ localtunnel 8080 #8080是你的端口，可以任意指定</p>
<p align="left">看到Port 8080 is now publicly accessible from http://8bv2.localtunnel.com …这样的信息表示成功启动</p>
<p align="left">由于localtunnel是基于ssh的，所以第一次执行的时候需要提供你的ssh public key：</p>
<p align="left">$ localtunnel -k ~/.ssh/id_rsa.pub 8080 （public如何生成？问google吧）</p>
<p align="left">执行完以后貌似不用管提示什么error之类的，重新有执行下 localtunnel 8080即可</p>
<p align="left">四.配置</p>
<p align="left">打开你的settings.js，修改如下内容</p>
<p align="left">export IG_CLIENT_ID=YOUR_CLIENT_ID</p>
<p align="left">export IG_CLIENT_SECRET=YOUR_CLIENT_SECRET</p>
<p align="left">cliend_id 和 client_secret 在这里申请：<a href="http://instagram.com/developer/manage/" target="_blank">http://instagram.com/developer/manage/</a></p>
<p align="left">五.执行</p>
<p align="left">OK，现在所有东西都准备就绪，我们开始执行吧</p>
<p align="left">$redis-server conf/redis.conf (Realtime-Demo提供的redis.conf默认把daemonize yes屏蔽了，如果你想redis以守护进程的形式运行，可以把他打开)</p>
<p align="left">$localtunnel 3000</p>
<p align="left">$node server.js</p>
<p align="left">export IG_CALLBACK_HOST=http://xxx.localtunnel.com(xxx这个根据你启动localtunnel后显示的具体来)</p>
<p align="left">订阅</p>
<p align="left">curl -F “client_id=$IG_CLIENT_ID” \</p>
<p align="left">-F “client_secret=$IG_CLIENT_SECRET” \</p>
<p align="left">-F ‘object=geography’ \</p>
<p align="left">-F ‘aspect=media’ \</p>
<p align="left">-F ‘lat=37.761216 ‘ \</p>
<p align="left">-F ‘lng=-122.43953′ \</p>
<p align="left">-F ‘radius=5000′ \</p>
<p align="left">-F “callback_url=$IG_CALLBACK_HOST/callbacks/geo/san-francisco/” \</p>
<p align="left"><a href="https://api.instagram.com/v1/subscriptions" target="_blank">https://api.instagram.com/v1/subscriptions</a></p>
<p align="left">都执行完以后，你会看到在node server.js的终端上定时吐出数据（如果还没有，那可能还没有），这就是你要的数据嘛</p>
<p align="left">用你的浏览器访问http://localhost:3000 看是否有数据？</p>
<p>&nbsp;</p>
