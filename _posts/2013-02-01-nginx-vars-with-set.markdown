---
layout: post
status: publish
published: true
title: "从set指令分析nginx变量赋值"
author: dcshi
author_login: dcshi
author_email: dcshi@qq.com
excerpt: "本文章以set指令展开讨论，要理解nginx的变量机制远不止这些.源码是最好的文档，另也推荐这blog: <a href=\"http://lenky.info/2012/08/08/nginx%E5%8F%98%E9%87%8F%E6%9C%BA%E5%88%B6/\"
  target=\"_blank\">http://lenky.info/</a>\r\n\r\n下面两个set语法是我们经常用到的\r\n<pre>set $my_name
  \"dcshi\";\r\nset $my_name $arg_name;</pre>\r\n&nbsp;\r\n\r\n理解set指令的赋值原理，对于我们开发nginx模块时候，对变量、常量的赋值操作很有帮助：\r\n\r\nset指令允许把常量(set
  $my_name dcshi)、变量 (set $my_name $arg_name)、组合变量(set $my_name ${arg_name}_shi) 赋值给变量$val.\r\n\r\n&nbsp;\r\n\r\nset对应的实现函数为static
  char * ngx_http_rewrite_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);\r\n\r\n该函数用了栈的实现思想，以一个变量赋值操作为例，如
  set $my_name ${arg_name}_shi;\r\n\r\n1.  把需要的变量对应的code_t入栈，即 变量$arg_name 和 常量_shi
  对应的code_t入栈;\r\n\r\ncode_t是什么？代码里我们看到是通过ngx_http_script_start_code(...)函数创建一个code_t。code简单理解就是一个处理函数，详细的下面讨论\r\n\r\n2.
   把set指令对应的code_t入栈，其code对应的函数逻辑是完成变量赋值。\r\n\r\n&nbsp;\r\n\r\n栈定义在ngx_http_rewrite_loc_conf，定义如下:\r\n<pre>typedef
  struct {\r\n\r\nngx_array_t  *codes;         //此结构保存着code_t\r\n\r\nngx_uint_t  
   stack_size;\r\n\r\nngx_flag_t    log;\r\n\r\nngx_flag_t    uninitialized_variable_warn;\r\n\r\n}
  ngx_http_rewrite_loc_conf_t;</pre>\r\n&nbsp;\r\n"
wordpress_id: 192
wordpress_url: http://www.dcshi.com/?p=192
date: '2013-02-01 18:43:24 +0000'
date_gmt: '2013-02-01 18:43:24 +0000'
categories:
- nginx
tags: []
---
<p>本文章以set指令展开讨论，要理解nginx的变量机制远不止这些.源码是最好的文档，另也推荐这blog: <a href="http://lenky.info/2012/08/08/nginx%E5%8F%98%E9%87%8F%E6%9C%BA%E5%88%B6/" target="_blank">http://lenky.info/</a></p>
<p>下面两个set语法是我们经常用到的</p>
<pre>set $my_name "dcshi";
set $my_name $arg_name;</pre>
<p>&nbsp;</p>
<p>理解set指令的赋值原理，对于我们开发nginx模块时候，对变量、常量的赋值操作很有帮助：</p>
<p>set指令允许把常量(set $my_name dcshi)、变量 (set $my_name $arg_name)、组合变量(set $my_name ${arg_name}_shi) 赋值给变量$val.</p>
<p>&nbsp;</p>
<p>set对应的实现函数为static char * ngx_http_rewrite_set(ngx_conf_t *cf, ngx_command_t *cmd, void *conf);</p>
<p>该函数用了栈的实现思想，以一个变量赋值操作为例，如 set $my_name ${arg_name}_shi;</p>
<p>1.  把需要的变量对应的code_t入栈，即 变量$arg_name 和 常量_shi 对应的code_t入栈;</p>
<p>code_t是什么？代码里我们看到是通过ngx_http_script_start_code(...)函数创建一个code_t。code简单理解就是一个处理函数，详细的下面讨论</p>
<p>2.  把set指令对应的code_t入栈，其code对应的函数逻辑是完成变量赋值。</p>
<p>&nbsp;</p>
<p>栈定义在ngx_http_rewrite_loc_conf，定义如下:</p>
<pre>typedef struct {

ngx_array_t  *codes;         //此结构保存着code_t

ngx_uint_t    stack_size;

ngx_flag_t    log;

ngx_flag_t    uninitialized_variable_warn;

} ngx_http_rewrite_loc_conf_t;</pre>
<p>&nbsp;<br />
<a id="more"></a><a id="more-192"></a><br />
<strong>set $val dcshi</strong> 对应的栈结构：<br />
<a href="http://www.dcshi.com/wp-content/uploads/2013/02/Snip20130202_3.png"><img class="alignleft size-full wp-image-213" title="Snip20130202_3" src="http://www.dcshi.com/wp-content/uploads/2013/02/Snip20130202_3.png" alt="" width="779" height="164" /></a></p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>&nbsp;</p>
<p>把上面的结构定义为栈有不恰当的地方，因为他是先进先出的，更应该是一个list；嗯，我们还是根据作者原意用stack来展开描述。</p>
<p>1.依次把常量(dcshi)和set指令的code_t入栈；</p>
<p>从上面可以看出，有多个不同类型的code_t，其对应的结构也不一样。但是他们会有一个共同的数据结构code. code对应一个函数指针.</p>
<p>2.依次(从左往右)出栈，并执行每个code_t对应的code函数。</p>
<p>什么时候出栈？在rewrite模块里可以看到出栈的操作在ngx_http_rewrite_handler这个函数里定义，同时我们知道这处理函数是注册在NGX_HTTP_SERVER_REWRITE_PHASE和NGX_HTTP_REWRITE_PHASE这两个运行阶段的。</p>
<p>&nbsp;</p>
<p>所以我们可以知道rewrite模块是把所有set指令初始化(即解析配置文件，把涉及的变量、常量依次入栈)，最后通过依次出栈，调用code_t.code完成所有set指令的赋值操作。</p>
<p>&nbsp;</p>
<p>我们来看看上面提到的两个code_t.code的逻辑。</p>
<p>1.常量code: ngx_http_script_value_code</p>
<pre>void ngx_http_script_value_code(ngx_http_script_engine_t *e)
{
   ngx_http_script_value_code_t  *code;

   /*
    * 获取常量的code_t
    * ngx_http_rewrite_handler(...):172行 e->ip指向rlcf->codes->elts; 即e->ip指向的上面提到的stack
    */
   code = (ngx_http_script_value_code_t *) e->ip;  

   /*
    * 指向stack的下一个单元，即模拟出栈操作
    */
   e->ip += sizeof(ngx_http_script_value_code_t);

   /*
    * e->sp是一个ngx_str_t类型的指针，指向(set $key $val)的val的值
    * 以 set $key dcshi; 为例, e->sp指向dcshi此常量字符串
    * 以 set $key ${arg1}_shi为例， e->sp指向组合变量的buf(nginx会临时分配一个buf，并把$arg1和_shi的数据copy在一起)
    * 所以，这里(指常量)e->sp 指向code->text_data，即指向常量字符串
    */
   e->sp->len = code->text_len;
   e->sp->data = (u_char *) code->text_data;

   /*
    * 下一个sp，指向NULL
    */
   e->sp++;
}</pre>
<p>&nbsp;</p>
<p>2.set指令code: ngx_http_script_set_var_code</p>
<pre>void ngx_http_script_set_var_code(ngx_http_script_engine_t *e)
{
   ngx_http_request_t          *r;
   ngx_http_script_var_code_t  *code;

   /*
    * 上面讨论过，指向当前code_t，即set指令的code_t
    */
   code = (ngx_http_script_var_code_t *) e->ip;
   e->ip += sizeof(ngx_http_script_var_code_t);

   /*
    * 我们知道变量是与请求相关的，例如remote_user等，每个请求都不同
    * 所以，变量的值，我们保存在reuest->variables里
    */
   r = e->request;
  
   /*
    * 上面讨论过，当前sp指向NULL，sp--指向上一个有效的数据单元
    */
   e->sp--;

   /*
    * code->index即code_t里定义的index字段，对应的是cmcf->variables索引数组的下标
    * r->variables[code->index].data = e->sp->data; 完成对变量的赋值
    * 这里有必要提下，nginx模块里取变量，很多时候通过ngx_http_get_indexed_variable(ngx_http_request_t *r, ngx_uint_t index)
    * r对应的请求结构，index就变量的索引。变量的索引可以通过ngx_http_get_variable_index(ngx_conf_t *cf, ngx_str_t *name)获取
    */
   r->variables[code->index].len = e->sp->len;
   r->variables[code->index].valid = 1;
   r->variables[code->index].no_cacheable = 0;
   r->variables[code->index].not_found = 0;
   r->variables[code->index].data = e->sp->data;
｝</pre>
<p>很明显，如果连续依次调用上面的这两个code，即可完成一个set指令的变量赋值。</p>
<p>&nbsp;</p>
<p>上面讨论的赋值常量，下面我们来看看赋值变量</p>
<p>set $val "${arg_xx}_shi";  #把变量$arg_xx和常量_shi拼接在一起，对$val赋值。</p>
<p><a href="http://www.dcshi.com/wp-content/uploads/2013/02/Snip20130202_4.png"><img class="alignleft size-full wp-image-220" title="Snip20130202_4" src="http://www.dcshi.com/wp-content/uploads/2013/02/Snip20130202_4.png" alt="" width="1103" height="436" /></a></p>
<p>&nbsp;</p>
<p>正如上面提到，出栈操作是从左往右执行；由于涉及变量，所以对应的栈结果有较之前复杂，但道理是一样的。</p>
<p>以这个set $key $val;为例，如果$val部分涉及nginx的变量(即带有$关键字)，我称他为复杂变量赋值</p>
<p>1.把一个固定的复杂变量出力code_t入栈，此code_t的逻辑是固定的，这也是与常量赋值最大的不同点；code_t里有一个lengths变量，实质也是一个栈结构。lengths栈保存的code_t是用来计算后续的复杂变量data的长度，以让nginx临时分配一个buf，以把复杂变量的data copy到buf.</p>
<p>2.把复杂变量里涉及的变量的code_t入栈，如set $val ${arg_xx}_shi;则涉及到两个code_t，分别为变量$arg_xx的code_t和常量(_shi)的code_t.当然他们的code_t.code处理逻辑是不一样的。</p>
<p>3.把set指令的code_t入栈。</p>
<p>4.同样地，如果把这个栈(含有四个单元)依次出栈，并调用相应的code_t.code，即可完成对$vval的赋值</p>
<p>&nbsp;</p>
<p>下面我们模拟出栈过程，一步步来弄理解set指令的赋值过程：</p>
<p>1.正如上述步骤1所提到的固定的复杂code_t. 其code处理函数为ngx_http_script_complex_value_code(...)。其主要完成两个事情</p>
<ul>
<li>扫描lengths stack，依次调用code_t.code计算后续的复杂变量data的长度。那lengths里的元素是谁插入的？详细请参考函数ngx_http_script_compile(ngx_http_script_compile_t *sc)｛...｝</li>
<li>根据步骤a计算得到的长度进行内存分配，并把e-&gt;pos指向buf</li>
</ul>
<p>2.调用ngx_http_script_copy_var_code，把$arg_xx的值copy到步骤1.b所分配的buf</p>
<p>3.调用ngx_http_script_copy_code，把常量_shi的值copy到步骤1.b所分配的buf</p>
<p>4.调用ngx_http_script_set_var_code，完成赋值操作。上文已经讨论。</p>
<p>&nbsp;</p>
<p>好吧，来总结下。</p>
<p>假设我们都了解set指令的stack工作原理，下面这两个set指令：</p>
<p><strong>set $v1 dcshi；</strong></p>
<p><strong>set $v2 "${arg_xx}_shi";</strong></p>
<p>&nbsp;</p>
<p>对应的stack结构为<br />
<a href="http://www.dcshi.com/wp-content/uploads/2013/02/Snip20130202_5.png"><img class="alignleft size-full wp-image-225" title="Snip20130202_5" src="http://www.dcshi.com/wp-content/uploads/2013/02/Snip20130202_5.png" alt="" width="976" height="91" /></a></p>
<p>所以依次（左-&gt;右）出栈，即可完成对$v1 $v2的赋值操作。</p>
<p>&nbsp;</p>
<p>文章转载请保留原文地址 <a href="http://www.dcshi.com/?p=192">http://www.dcshi.com/?p=192</a></p>
