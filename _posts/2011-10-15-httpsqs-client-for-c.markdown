---
layout: post
status: publish
published: true
title: 'httpsqs client for c '
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
wordpress_id: 46
wordpress_url: http://www.dcshi.com/?p=46
date: '2011-10-15 10:54:23 +0000'
date_gmt: '2011-10-15 10:54:23 +0000'
categories:
- linux
tags:
- httpsqs
---
<p>张宴老师（@rewinx）写的基于Tokyo cabinet的消息队列(<a href="http://blog.s135.com/httpsqs_1_2/" target="_blank">http simple queue server-httpsqs</a> )，关于httsqs的介绍，请移步！<br />
关于httpsqs张老师只提供了php与perl的client，这两天用了下班后自己写了一个c的client，code：<br />
<a href="http://github.com/dcshi/httpsqs_client_for_c/downloads" target="_blank">download</a><br />
<code>/*<br />
*HTTP Simple Queue Service -httpsqs client for C<br />
*<br />
*Author: dcshi(twitter:@dcshi) E-mail:dcshi@qq.com<br />
*This is free SoftWare,and you are welcome to modify and redistribute it under the New BSD License<br />
*<br />
* http connect without Keep-Alive<br />
*-------------------------------------------------------------------------------<br />
* char* get(char* srcIp,int port,char* charset,char* queue_name);<br />
* char* put(char* srcIp,int port,char* charset,char* queue_name,char* data);<br />
* char* view(char* srcIp,int port,char* charset,char* queue_name,int pos);<br />
* char* reset(char* srcIp,int port,char* charset,char* queue_name);<br />
* char* status(char* srcIp,int port,char* charset,char* queue_name);<br />
* char* maxqueue(char* srcIp,int port,char* charset,char* queue_name,int num);<br />
*--------------------------------------------------------------------------------<br />
*<br />
* http pconnect with Keep-Alive (Very fast in PHP FastCGI mode &amp; Command line mode)<br />
*--------------------------------------------------------------------------------<br />
* char* pget(char* srcIp,int port,char* charset,char* queue_name);<br />
* char* pput(char* srcIp,int port,char* charset,char* queue_name,char* data);<br />
* char* pview(char* srcIp,int port,char* charset,char* queue_name,int pos);<br />
* char* preset(char* srcIp,int port,char* charset,char* queue_name);<br />
* char* pstatus(char* srcIp,int port,char* charset,char* queue_name);<br />
* char* pmaxqueue(char* srcIp,int port,char* charset,char* queue_name,int num);<br />
*--------------------------------------------------------------------------------<br />
*/</code></p>
<p>#include<br />
#include<br />
#include<br />
#include<br />
#include<br />
#include<br />
#include<br />
#include<br />
#include<br />
#include<br />
#include<br />
#include<br />
#include<br />
#include</p>
<p>int sockfd;<br />
char hostIp[16];<br />
int hPort;<br />
int ret;<br />
struct sockaddr_in server_addr;<br />
char buffer[512];</p>
<p>int init(char* srcIp,int port)<br />
{<br />
memset(hostIp,0,sizeof(hostIp));<br />
memcpy(hostIp,srcIp,strlen(srcIp));<br />
hPort = port;<br />
sockfd = socket(AF_INET,SOCK_STREAM,0);<br />
if(sockfd == -1)<br />
exit(1);<br />
bzero(&amp;server_addr,sizeof(server_addr));<br />
server_addr.sin_family = AF_INET;<br />
server_addr.sin_port = htons(hPort);<br />
server_addr.sin_addr.s_addr=inet_addr(hostIp);<br />
ret = hConnect();<br />
return ret;<br />
}</p>
<p>int hConnect()<br />
{<br />
ret = connect(sockfd,(struct sockaddr*)(&amp;server_addr),sizeof(struct sockaddr));<br />
if(ret == -1)<br />
{<br />
printf("Connect Error:%s\a\n",strerror(errno));<br />
exit(-1);<br />
}<br />
return ret;</p>
<p>}</p>
<p>int http_post(char* query,char* data)<br />
{<br />
char request[1024];<br />
int nbytes = 0;<br />
int send=0;<br />
int totalsend=0;<br />
int datalen;</p>
<p>datalen = strlen(data);<br />
memset(request,0,sizeof(request));<br />
sprintf(request, "POST %s HTTP/1.1\r\nHost: %s\r\nContent-Length:%d\r\nConnection: close\r\n\r\n%s", query, hostIp, datalen,data);<br />
nbytes = strlen(request);<br />
while(totalsend &lt; nbytes)<br />
{<br />
send = write(sockfd,request+totalsend,nbytes-totalsend);<br />
totalsend += send;<br />
}</p>
<p>memset(buffer,0,sizeof(buffer));<br />
nbytes = read(sockfd,buffer,sizeof(buffer));<br />
close(sockfd);</p>
<p>char* buf = strtok(buffer,"\r\n");<br />
int pos = -1;<br />
while(buf != NULL)<br />
{<br />
if(strstr(buf,"Pos:"))<br />
{<br />
pos = atoi(buf+5);<br />
break;<br />
}<br />
buf = strtok(NULL,"\r\n");<br />
}<br />
return pos;</p>
<p>}</p>
<p>char* http_get(char* query)<br />
{<br />
char request[1024];<br />
int nbytes = 0;<br />
int send=0;<br />
int totalsend=0;</p>
<p>memset(request,0,sizeof(request));<br />
sprintf(request, "GET %s HTTP/1.1\r\nHost: %s\r\nConnection: close\r\n\r\n", query, hostIp );<br />
nbytes = strlen(request);<br />
while(totalsend &lt; nbytes)<br />
{<br />
send = write(sockfd,request+totalsend,nbytes-totalsend);<br />
totalsend += send;<br />
}</p>
<p>memset(buffer,0,sizeof(buffer));<br />
nbytes = read(sockfd,buffer,sizeof(buffer));<br />
close(sockfd);<br />
char* ptr = strstr(buffer,"\r\n\r\n");<br />
ptr = ptr + 4;<br />
return ptr;<br />
}</p>
<p>int http_ppost(char* query,char *data)<br />
{<br />
char request[1024];<br />
int nbytes = 0;<br />
int send=0;<br />
int totalsend=0;<br />
int datalen;</p>
<p>datalen = strlen(data);<br />
memset(request,0,sizeof(request));<br />
sprintf(request, "POST %s HTTP/1.1\r\nHost: %s\r\nContent-Length:%d\r\nConnection: Keep-Alive\r\n\r\n%s", query, hostIp, datalen,<br />
data);<br />
nbytes = strlen(request);<br />
while(totalsend &lt; nbytes)<br />
{<br />
send = write(sockfd,request+totalsend,nbytes-totalsend);<br />
totalsend += send;<br />
}</p>
<p>memset(buffer,0,sizeof(buffer));<br />
nbytes = read(sockfd,buffer,sizeof(buffer));<br />
close(sockfd);</p>
<p>char* buf = strtok(buffer,"\r\n");<br />
int pos = -1;<br />
while(buf != NULL)<br />
{<br />
if(strstr(buf,"Pos:"))<br />
{<br />
pos = atoi(buf+5);<br />
break;<br />
}<br />
buf = strtok(NULL,"\r\n");<br />
}<br />
return pos;</p>
<p>}</p>
<p>char* http_pget(char* query)<br />
{<br />
char request[1024];<br />
int nbytes = 0;<br />
int send=0;<br />
int totalsend=0;</p>
<p>memset(request,0,sizeof(request));<br />
sprintf(request, "GET %s HTTP/1.1\r\nHost: %s\r\nConnection: Keep-Alive\r\n\r\n", query, hostIp );<br />
nbytes = strlen(request);<br />
while(totalsend &lt; nbytes)<br />
{<br />
send = write(sockfd,request+totalsend,nbytes-totalsend);<br />
totalsend += send;<br />
}</p>
<p>memset(buffer,0,sizeof(buffer));<br />
nbytes = read(sockfd,buffer,sizeof(buffer));<br />
close(sockfd);<br />
char* ptr = strstr(buffer,"\r\n\r\n");<br />
ptr = ptr + 4;<br />
return ptr;<br />
}</p>
<p>int put(char* srcIp,int port,char* charset,char* queue_name,char* data)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=put",charset,queue_name);<br />
return http_post(query,data);<br />
}</p>
<p>char* get(char* srcIp,int port,char* charset,char* queue_name)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=get",charset,queue_name);<br />
return http_get(query);<br />
}</p>
<p>int pput(char* srcIp,int port,char* charset,char* queue_name,char* data)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=put",charset,queue_name);<br />
return http_ppost(query,data);<br />
}</p>
<p>char* pget(char* srcIp,int port,char* charset,char* queue_name)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=get",charset,queue_name);<br />
return http_pget(query);<br />
}</p>
<p>char* status(char* srcIp,int port,char* charset,char* queue_name)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=status",charset,queue_name);<br />
return http_get(query);<br />
}</p>
<p>char* reset(char* srcIp,int port,char* charset,char* queue_name)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=reset",charset,queue_name);<br />
return http_get(query);<br />
}</p>
<p>char* view(char* srcIp,int port,char* charset,char* queue_name,int pos)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=view&amp;pos=%d",charset,queue_name,pos);<br />
return http_get(query);<br />
}</p>
<p>char* maxqueue(char* srcIp,int port,char* charset,char* queue_name,int num)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=maxqueue&amp;num=%d",charset,queue_name,num);<br />
return http_get(query);<br />
}</p>
<p>char* pstatus(char* srcIp,int port,char* charset,char* queue_name)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=status",charset,queue_name);<br />
return http_pget(query);<br />
}</p>
<p>char* preset(char* srcIp,int port,char* charset,char* queue_name)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=reset",charset,queue_name);<br />
return http_pget(query);<br />
}</p>
<p>char* pview(char* srcIp,int port,char* charset,char* queue_name,int pos)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=view&amp;pos=%d",charset,queue_name,pos);<br />
return http_pget(query);<br />
}</p>
<p>char* pmaxqueue(char* srcIp,int port,char* charset,char* queue_name,int num)<br />
{<br />
char query[256];<br />
init(srcIp,port);<br />
memset(query,0,sizeof(query));<br />
sprintf(query,"/?charset=%s&amp;name=%s&amp;opt=maxqueue&amp;num=%d",charset,queue_name,num);<br />
return http_pget(query);<br />
}</p>
