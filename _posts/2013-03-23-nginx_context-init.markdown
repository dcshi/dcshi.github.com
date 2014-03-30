---
layout: post
status: publish
published: true
title: nginx 上下文结构初始化流程
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
excerpt: "说在前面：\r\n\r\n本文的讨论基于的nginx版本是1.2.1。\r\n\r\n文章对于nginx的一些背景知识（如什么是模块，上下文等）没有过多提及，本文适合对nginx有一定了解的同学。\r\n\r\n&nbsp;\r\n\r\nngx_conf_t
  *cf; 做过nginx模块开发，或者是有学习过nginx源码的同学会对这个结构体非常眼熟了。\r\n\r\nngx_cycle.c:251 conf.ctx
  = cycle->conf_ctx;\r\n\r\n&nbsp;\r\n\r\nctx(context)在nginx里随处可见，字面理解是上下文；在nginx里，ctx保存的每个模块的配置结构.\r\n\r\ncycle->conf_ctx是一个四重指针，为什么需要四重指针？希望读完本文，会对你理解有一定帮助。\r\n\r\n&nbsp;\r\n\r\n&nbsp;\r\n\r\n<strong>nginx配置文件nginx.conf</strong>（文章会围绕这个配置文件，分析nginx解析并保存模块上下文的过程）\r\n<pre>1
  http {\r\n2     resolver 8.8.8.8;\r\n3\r\n4     server\r\n5     {\r\n6         listen
  80;\r\n7         server_name dcshi.com\r\n8         root /data/html\r\n9\r\n10         location
  /a\r\n11         {\r\n12             set $foo $bar;\r\n13\r\n14             return
  200;\r\n15         }\r\n16\r\n17         location /b.html\r\n18         {\r\n19
  \            root /opt/html;\r\n20         }\r\n21\r\n22     }\r\n23\r\n24     server\r\n25
  \    {\r\n26         listen 80;\r\n27         server_name s2.dcshi.com\r\n28\r\n29
  \        location /a {}\r\n30\r\n31         location /b {}\r\n32     }\r\n33 }</pre>\r\n&nbsp;\r\n\r\n<strong>与上述配置文件对应的模块上下文(ctx)架构图:</strong>\r\n\r\n<a
  href=\"http://www.dcshi.com/wp-content/uploads/2013/03/nginx_ctx4.jpeg\"><img class=\"alignleft
  size-full wp-image-337\" title=\"nginx_ctx\" src=\"http://www.dcshi.com/wp-content/uploads/2013/03/nginx_ctx4.jpeg\"
  alt=\"\" width=\"963\" height=\"677\" /></a>\r\n"
wordpress_id: 333
wordpress_url: http://www.dcshi.com/?p=333
date: '2013-03-23 18:28:42 +0000'
date_gmt: '2013-03-23 18:28:42 +0000'
categories:
- nginx
tags: 
- nginx
---
<p>说在前面：</p>
<p>本文的讨论基于的nginx版本是1.2.1。</p>
<p>文章对于nginx的一些背景知识（如什么是模块，上下文等）没有过多提及，本文适合对nginx有一定了解的同学。</p>
<p>&nbsp;</p>
<p>ngx_conf_t *cf; 做过nginx模块开发，或者是有学习过nginx源码的同学会对这个结构体非常眼熟了。</p>
<p>ngx_cycle.c:251 conf.ctx = cycle->conf_ctx;</p>
<p>&nbsp;</p>
<p>ctx(context)在nginx里随处可见，字面理解是上下文；在nginx里，ctx保存的每个模块的配置结构.</p>
<p>cycle->conf_ctx是一个四重指针，为什么需要四重指针？希望读完本文，会对你理解有一定帮助。</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p><strong>nginx配置文件nginx.conf</strong>（文章会围绕这个配置文件，分析nginx解析并保存模块上下文的过程）</p>
<pre>1 http {
2     resolver 8.8.8.8;
3
4     server
5     {
6         listen 80;
7         server_name dcshi.com
8         root /data/html
9
10         location /a
11         {
12             set $foo $bar;
13
14             return 200;
15         }
16
17         location /b.html
18         {
19             root /opt/html;
20         }
21
22     }
23
24     server
25     {
26         listen 80;
27         server_name s2.dcshi.com
28
29         location /a {}
30
31         location /b {}
32     }
33 }</pre>
<p>&nbsp;</p>
<p><strong>与上述配置文件对应的模块上下文(ctx)架构图:</strong></p>
<p><a href="http://www.dcshi.com/wp-content/uploads/2013/03/nginx_ctx4.jpeg"><img class="alignleft size-full wp-image-337" title="nginx_ctx" src="http://www.dcshi.com/wp-content/uploads/2013/03/nginx_ctx4.jpeg" alt="" width="963" height="677" /></a><br />
<a id="more"></a><a id="more-333"></a><br />
1）ngx_cycle.c</p>
215     for (i = 0; ngx_modules\[i\]; i++) {
216         if (ngx_modules\[i\]->type != NGX_CORE_MODULE) {
217             continue;
218         }
219
220         module = ngx_modules\[i\]->ctx;

//如果是核心模块，而且有定义了create_conf，则调用模块的create_conf来创建上下文结构
222         if (module->create_conf) {
223             rv = module->create_conf(cycle);
224             if (rv == NULL) {
225                 ngx_destroy_pool(pool);
226                 return NULL;
227             }
228             cycle->conf_ctx\[ngx_modules\[i\]->index\] = rv;
229         }
230     }
<p>&nbsp;</p>
<p>2）ngx_cycle.c</p>
<pre>/*
* 调取ngx_conf_parse开始解析配置文件(nginx.conf),ngx_conf_parse是一个很通用的函数。
* 参数二是一个文件名，如果传入NULL，则解析当前的block配置。{ }配置文件里两花括号定义一个block。
* /
268     if (ngx_conf_parse(&amp;conf, &amp;cycle->conf_file) != NGX_CONF_OK) {
269         environ = senv;
270         ngx_destroy_cycle_pools(&amp;conf);
271         return NULL;
272     }</pre>
<p>&nbsp;</p>
<p>3）ngx_conf_file.c：</p>
<pre>/*
* 如果指令指定了handler函数，则直接调用
* cf->handler 什么时候不为NULL？后续再讨论。
*/
222         if (cf->handler) {
223
224             /*
225              * the custom handler, i.e., that is used in the http's
226              * "types { ... }" directive
227              */
228
229             rv = (*cf->handler)(cf, NULL, cf->handler_conf);
230             if (rv == NGX_CONF_OK) {
231                 continue;
232             }
233
234             if (rv == NGX_CONF_ERROR) {
235                 goto failed;
236             }
237
238             ngx_conf_log_error(NGX_LOG_EMERG, cf, 0, rv);
239
240             goto failed;
241         }
242
243
// cf->hanlder为NULL，则调用ngx_conf_handler来遍历所有模块的指令，以触发该指令对应的回调函数。
244         rc = ngx_conf_handler(cf, rc);</pre>
<p>&nbsp;</p>
<p>4）ngx_conf_file.c</p>
<pre>/*
* NGX_DIRECT_CONF为什么是direct(直接)，这类命令字只出现在core模块里
* direct字面理解是需要使用（赋值）时候可以直接拿来使用即可，即在调用前上下文结构已经分配好，如worker_processes指令
* 对应的非direct则需要先分配空间后才能使用，如http指令(http是一个指令，不是一类指令，不要搞混哦！)
* 上面提到的步骤一，nginx在解析配置文件之前，会依次调用NGX_CORE_MODULE 模块的create_conf，
* direct指令的上下文就是这个时候分配的.
*
*/
380             if (cmd->type & NGX_DIRECT_CONF) {
381                 conf = ((void **) cf->ctx)\[ngx_modules\[i\]->index\];
382
/*
* http指令就是一个NGX_MAIN_CONF指令(怎么确定指令type，请看ngx_command_s定义)
* 细心的同学看到，381行与384行的唯一区别是，后者是返回指针的地址。
* c/c++开发的同学都知道 func(char *p) 与 func(char **p)的区别。后者的func可以对传入的p重新分配内存
* 如http指令为例，http所在模块是ngx_http_module，其没有定义create_conf(ngx_http.c:98)
* 即在步骤一里是不会给ngx_http_module分配上下文结构的，所以需要在ngx_http_block里进行动态分配
* 所以，这就是为什么需要传入的是指针的地址。
* 至于为什么ngx_http_module不也提供create_conf回调，理论上结构更统一？答案留给你们自己思考。
*/
383             } else if (cmd->type & NGX_MAIN_CONF) {
384                 conf = &(((void **) cf->ctx)\[ngx_modules\[i\]->index\]);
385
386             } else if (cf->ctx) {
/*
* cmd->conf是什么？set指令的conf是 NGX_HTTP_LOC_CONF_OFFSET(ngx_http_rewrite_module.c:81)
* #define NGX_HTTP_LOC_CONF_OFFSET   offsetof(ngx_http_conf_ctx_t, loc_conf)
* 通过上面的宏定义，清楚可以知道cmd->conf返回的是一个变量在struct里的偏移
* nginx为每个http模块提供的上下文结构(ngx_http_conf_ctx_t)包括main_conf, srv_conf, loc_conf 三个元素；
* 对于一个指令而言，只会用到上下文结构中的一个*_conf；但对于一个模块（包含多个指令）而言，可能会用到任意多个*_conf。
* set指令可以运行在location和server，set指令的上下文结构又保存在srv_conf or loc_conf？请独立思考来验证自己是否已经对模块有一定理解。
*/
387                 confp = *(void **) ((char *) cf->ctx + cmd->conf);
388
389                 if (confp) {
390                     conf = confp\[ngx_modules\[i\]->ctx_index\];
391                 }
392             }
393
/* 调用指令对应的回调函数进行初始化操作 */
394             rv = cmd->set(cf, cmd, conf);</pre>
<p>&nbsp;</p>
<p>通过上面的代码片段，我们可以理解到，nginx是在边解析配置文件，边查找对应的模块指令，并调用指令绑定的回调函数进行初始化。<br />
下面是http指令的定义:</p>
<pre>83 static ngx_command_t  ngx_http_commands\[\] = {
84
85     { ngx_string("http"),
86       NGX_MAIN_CONF|NGX_CONF_BLOCK|NGX_CONF_NOARGS,
87       ngx_http_block,
88       0,
89       0,
90       NULL },
91
92       ngx_null_command
93 };</pre>
<p>ngx_http_block是一个很重要函数，当nginx在解析配置文件阶段，读取到http指令，即调用ngx_http_block函数对 http {….} 花括号里的配置进行解析。</p>
<p>&nbsp;</p>
<p>5）ngx_http.c</p>
<pre>120 ngx_http_block(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
121 {
…...

130
131     /* the main http context */
132
133     ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
134     if (ctx == NULL) {
135         return NGX_CONF_ERROR;
136     }
137
/*

* 注意，传进来的conf被修改了。

* 因为http_module虽是core模块，但是并没有定义create_conf，所以步骤一并没有为http_module分配上下文空间

* http指令是NGX_MAIN_CONF，所以conf传入的实质是一个指针的指针

* 所以通过在这里重新赋值，来为conf分配空间

*/
138     *(ngx_http_conf_ctx_t **) conf = ctx;
139
140
141     /* 为所有的http module计算ctx_index */
142
143     ngx_http_max_module = 0;
144     for (m = 0; ngx_modules\[m\]; m++) {
145         if (ngx_modules\[m\]->type != NGX_HTTP_MODULE) {
146             continue;
147         }
148
149         ngx_modules\[m\]->ctx_index = ngx_http_max_module++;
150     }
151
152
/*
* 为http ctx的的main_conf, srv_conf loc_conf 分配内存；
* 可以看到每个*_conf都是定长的,长度为ngx_http_max_module，
* 即每个模块都会被分配main_conf, srv_conf loc_conf，而不管改模块是否需要。
*/
155     ctx->main_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
…...
167     ctx->srv_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
…….
178     ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
/* 遍历所有的模块，分别调用其回调函数进行初始化 */
189     for (m = 0; ngx_modules\[m\]; m++) {
190         if (ngx_modules\[m\]->type != NGX_HTTP_MODULE) {
191             continue;
192         }
193
194         module = ngx_modules\[m\]->ctx;
195         mi = ngx_modules\[m\]->ctx_index;
196
197         if (module->create_main_conf) {
198             ctx->main_conf\[mi\] = module->create_main_conf(cf);
202         }
203
204         if (module->create_srv_conf) {
205             ctx->srv_conf\[mi\] = module->create_srv_conf(cf);
209         }
210
211         if (module->create_loc_conf) {
212             ctx->loc_conf\[mi\] = module->create_loc_conf(cf);
216         }
217     }
218
/* 对cf进行一个临时拷贝，在329行重新恢复。 */
219     pcf = *cf;
/*
* 注意：改变cf->ctx的上下文；nginx在解析配置文件的过程中，cf->ctx总会临时修改（最后被恢复）。
* 在解析http {} 的时候cf->ctx表示http_module的上下文结构
* 在解析server{} 的时候cf->ctx表示的是server的上下文结构
* 在解析location{} 的时候，cf->ctx表示的是location的上下文结构
* 可结合文章开头的架构图理解
*/
220     cf->ctx = ctx;
…...
235
236     /* 解析http { }, 依次调用涉及的指令回调函数，server作为http里一个必须的指令，也是在这个时候被调用  */
238     cf->module_type = NGX_HTTP_MODULE;
239     cf->cmd_type = NGX_HTTP_MAIN_CONF;
240     rv = ngx_conf_parse(cf, NULL);
241
242     if (rv != NGX_CONF_OK) {
243         goto failed;
244     }
…..
328
329     *cf = pcf;</pre>
<p>&nbsp;</p>
<p>6）ngx_http_core_module.c</p>
<pre>/*
* 注意：正如第三个参数的命名-dummy(假的)，本函数里没有使用.
* 没有使用传入的conf，那http与server的ctx是怎样串联的？(答案在文章开篇的架构图)
*/
2832 ngx_http_core_server(ngx_conf_t *cf, ngx_command_t *cmd, void *dummy)
2833 {
……
/* 为server指令分配一个上下文，这个上文终究要跟http的上下文串联起来的 */
2845     ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
2846     if (ctx == NULL) {
2847         return NGX_CONF_ERROR;
2848     }
/*
* 注意：server的ctx->main_conf是直接指向http_ctx（http的ctx，比server ctx更高一级）的main_conf
* 与ngx_http_block不同，这里只对上下文结构里的srv_conf和loc_conf进行内存分配
* 原因很简单，在server里调用的指令不可能用到main_conf的配置结构.
* 相同的道理，location里的指令不可能用到main_conf和srv_conf，所以只需要对loc_conf内存即可(ngx_http_core_location)
* 建议结合文章开篇的架构图理解
*/
2850     http_ctx = cf->ctx;
2851     ctx->main_conf = http_ctx->main_conf;
2852
…….
2855     ctx->srv_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
…….
2862     ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
/* 遍历所有的http module，对分别调用create_srv_conf 和 create_loc_conf进行初始化上下文配置 */
2867     for (i = 0; ngx_modules\[i\]; i++) {
2868         if (ngx_modules\[i\]->type != NGX_HTTP_MODULE) {
2869             continue;
2870         }
2871
2872         module = ngx_modules\[i\]->ctx;
2873
2874         if (module->create_srv_conf) {
2875             mconf = module->create_srv_conf(cf);
2880             ctx->srv_conf\[ngx_modules\[i\]->ctx_index\] = mconf;
2881         }
2882
2883         if (module->create_loc_conf) {
2884             mconf = module->create_loc_conf(cf);
2889             ctx->loc_conf\[ngx_modules\[i\]->ctx_index\] = mconf;
2890         }
2891     }
/*
* 核心来了，这段代码解释了http_ctx与srv_ctx怎样串联
* 2896, 2897行目的是把整个srv_ctx 先保存在ngx_http_core_module模块的ctx变量里
* 结合架构图理解
*/
2896     cscf = ctx->srv_conf\[ngx_http_core_module.ctx_index\];
2897     cscf->ctx = ctx;
2898
/*
* 串联了，http_ctx->main_conf(与srv_ctx->main_conf是等价的，还记得2851行吧?)
* 2900 - 2907行展示了串联的逻辑：整个ctx并没有直接串联，实际上是通过ngx_http_core_module模块进行串联的
* cmcf->servers是一个array类型，所以nginx.conf里配置的多个server也被依次串联起来
* 结合架构图理解
* /
2900     cmcf = ctx->main_conf\[ngx_http_core_module.ctx_index\];
2902     cscfp = ngx_array_push(&cmcf->servers);
2903     if (cscfp == NULL) {
2904         return NGX_CONF_ERROR;
2905     }
2906
2907     *cscfp = cscf;
2908
/* 继续调用ngx_conf_parse对server{ } 进行解析，其中location作为server里的一个指令，也在这个时候被调用。见步骤7 */
2911
2912     pcf = *cf;
2913     cf->ctx = ctx;
2914     cf->cmd_type = NGX_HTTP_SRV_CONF;
2915
2916     rv = ngx_conf_parse(cf, NULL);
2917

/* cf被恢复 */
2918     *cf = pcf;</pre>
<p>&nbsp;</p>
<p>7) ngx_http_core_module.c</p>
<pre>/*
* 注意：正如第三个参数的命名-dummy(假的)，本函数里没有使用.
*/
2956 ngx_http_core_location(ngx_conf_t *cf, ngx_command_t *cmd, void *dummy)
2957 {
…….
2968     ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_conf_ctx_t));
2969     if (ctx == NULL) {
2970         return NGX_CONF_ERROR;
2971     }
2972
/* 正如上文所说，location里的指令不可能使用main_conf和srv_conf，所以只需要为loc_conf分配空间 */
2973     pctx = cf->ctx;
2974     ctx->main_conf = pctx->main_conf;
2975     ctx->srv_conf = pctx->srv_conf;
2976
2977     ctx->loc_conf = ngx_pcalloc(cf->pool, sizeof(void *) * ngx_http_max_module);
/* 变量所有http module，依次调用create_loc_conf进行上下文初始化操作 */
2982     for (i = 0; ngx_modules\[i\]; i++) {
2983         if (ngx_modules\[i\]->type != NGX_HTTP_MODULE) {
2984             continue;
2985         }

2987         module = ngx_modules\[i\]->ctx;
2988
2989         if (module->create_loc_conf) {
2990             ctx->loc_conf\[ngx_modules\[i\]->ctx_index\] = module->create_loc_conf(cf);
2996     }
2997
/*
* 2998 - 3084 与步骤6类似，通过ngx_http_core_module 进行串联
* 结合架构图理解
*/
2998     clcf = ctx->loc_conf\[ngx_http_core_module.ctx_index\];
2999     clcf->loc_conf = ctx->loc_conf;
3000
…….
3083     pclcf = pctx->loc_conf\[ngx_http_core_module.ctx_index\];
3084
3085     if (pclcf->name.len) {
……. //location是允许嵌套的，如location /a { location /b { } } ,暂不讨论
3131     }
3132
3133     if (ngx_http_add_location(cf, &pclcf->locations, clcf) != NGX_OK) {
3134         return NGX_CONF_ERROR;
3135     }
/*  继续对location里的指令进行解析；值得注意的是，cf->ctx被修改，指向的是当前location的ctx */

3137     save = *cf;
3138     cf->ctx = ctx;
3139     cf->cmd_type = NGX_HTTP_LOC_CONF;
3140
3141     rv = ngx_conf_parse(cf, NULL);
3142
3143     *cf = save;
3144
3145     return rv;
3146 }</pre>
<p>&nbsp;</p>
<p>8）ngx_http.c<br />
步骤5, http对应的回调函数是ngx_http_block.在240行完成对整个http { } block的解析以后，nginx还需要合并配置。<br />
从上面的分析，提及到的ctx有http_ctx, srv_ctx, loc_ctx；以loc_conf为例，三个ctx都有包含，而且是互相独立的。<br />
如 root 指令既可以定义在server { } ，也可以定义在location { };<br />
如果location里没有对root进行定义，则location的loc_conf需要继承server { } 的root配置，这就是merge的作用。<br />
下面的代码则展示了，nginx是如何进行merge：</p>
<ol>
<li>分别 merge http_ctx 和 srv_ctx的 srv_conf 和 loc_conf</li>
<li>merge srv_ctx 和 loc_ctx的loc_conf</li>
</ol>
<pre>251     cmcf = ctx->main_conf\[ngx_http_core_module.ctx_index\];
252     cscfp = cmcf->servers.elts;
253
254     for (m = 0; ngx_modules\[m\]; m++) {
255         if (ngx_modules\[m\]->type != NGX_HTTP_MODULE) {
256             continue;
257         }
258
259         module = ngx_modules\[m\]->ctx;
260         mi = ngx_modules\[m\]->ctx_index;
261
262         /* init http{} main_conf's */
263
264         if (module->init_main_conf) {
265             rv = module->init_main_conf(cf, ctx->main_conf\[mi\]);
266             if (rv != NGX_CONF_OK) {
267                 goto failed;
268             }
269         }
270
271         rv = ngx_http_merge_servers(cf, cmcf, module, mi);
272         if (rv != NGX_CONF_OK) {
273             goto failed;
274         }
275     }</pre>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>祝玩得开心！</p>
<p>&nbsp;</p>
