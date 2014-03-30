---
layout: post
status: publish
published: true
title: nodejs+socketio+redis的一些小尝试
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 21
wordpress_url: http://www.dcshi.com/?p=21
date: '2011-10-15 10:29:30 +0000'
date_gmt: '2011-10-15 10:29:30 +0000'
categories:
- nodejs
tags:
- nodejs
---
<p>nodejs全异步的特点正的非常适合做聊天室，或者基于BS架构的即时通讯类的产品的开发</p>
<p>nodejs的详细介绍见：http://nodejs.org/</p>
<p>socket.io:众所周知现在html5火，html5的新特性包括本地存储，geo，webscoket等<br />
那socket.io有什么关系？答案是没有关系。<br />
socket.io分server&amp;client两部分，基于socket.io就可以完成一个简单的聊天室。socket.io是对client端的连接方式进行了封装，而且提供了很多强大的特性，如触发事件等<br />
由于不是所有的browser都支持webscoket，所以socket.io封装了以下几种连接模式，按优先级来决定采用哪种连接方式<br />
['websocket', 'server-events', 'flashsocket', 'htmlfile', 'xhr-multipart', 'xhr-polling']<br />
详细介绍就按：http://socket.io</p>
<p>redis，redis是一个高效的内存key-val存储系统，这里更多的是采用redis的pub/sub特性<br />
详细见redis.io</p>
<p>我主要是打算利用三者做一个基于BS架构的实时监控系统（如果你有兴趣，可以延伸成其他任何的实时系统，不只是监控，例如用户的实时行为监控系统？非常有趣）</p>
<p><strong>code for server:</strong><br />
var redis = require(‘redis’),<br />
settings = require(‘./settings’),<br />
parser = require(‘./parser’),<br />
helpers = require(‘./helpers’);<br />
subscriptionPattern = ‘channel:*’,<br />
io = require(’socket.io’);</p>
<p>var app = settings.app;<br />
app.listen(settings.appPort);</p>
<p>var server = io.listen(app);<br />
helpers.debug(‘Server running at http://127.0.0.1:’+settings.appPort);</p>
<p>app.get(‘/’, function(request, response){<br />
response.render(‘index’, {<br />
title: ‘test’,<br />
address: ‘http://192.168.39.2′,<br />
port: 1313<br />
});<br />
});</p>
<p>var redisClient = redis.createClient(settings.REDIS_PORT, settings.REDIS_HOST);<br />
server.sockets.on(‘connection’, function (socket) {<br />
socket.on(’subscribe’,function(channel)<br />
{<br />
helpers.debug(‘join channel: ‘+channel);<br />
socket.join(channel);<br />
socket.send(’subscribe OK’);<br />
});</p>
<p>socket.on(‘unsubscribe’,function(channel)<br />
{<br />
helpers.debug(‘leave channel: ‘+channel);<br />
socket.leave(channel);<br />
socket.send(‘unsubscribe OK’);<br />
});</p>
<p>});</p>
<p>redisClient.psubscribe(subscriptionPattern);<br />
redisClient.on(‘pmessage’,function(pattern, channel, message){<br />
var channelName = ”;<br />
helpers.debug(‘pattern: ‘+pattern);<br />
if(pattern == subscriptionPattern)<br />
{<br />
try<br />
{<br />
channelName = channel.split(‘:’)[1].replace(/-/g, ‘ ‘);<br />
}<br />
catch(e){return;}<br />
}</p>
<p>helpers.debug(‘message:\n ‘+message);<br />
server.store.clients(channelName,function(clients){<br />
helpers.debug(‘channel: ‘+channelName);<br />
clients.forEach(function(id){<br />
helpers.debug(‘client id: ‘+id);<br />
var packet = {<br />
type: ‘message’,<br />
data: message<br />
};<br />
packet = parser.encodePacket(packet)<br />
server.store.client(id).publish(packet);<br />
});<br />
})<br />
});</p>
<p><strong>code for client:</strong><br />
script(src=’./socket.io/socket.io.js’)<br />
script(src=’http://staging.tokbox.com/v0.91/js/TB.min.js’)<br />
script(src=’./javascripts/app.js’)<br />
var socket = io.connect(‘htpp://serverip:port’);<br />
socket.emit(“subscribe”,”channel-test”);</p>
<p>socket.on(‘message’, function (data) {<br />
console.log(data);</p>
<p>});</p>
<p>上面只提供了一个实现思想，有兴趣的同学自己琢磨下吧<br />
详细代码见：</p>
<p>https://github.com/downloads/dcshi/node-realtime-framework/nodejs_socket.io_redis.tar</p>
<p>&nbsp;</p>
