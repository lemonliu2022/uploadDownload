**实现一个可以任意扩展的、上传下载文件的服务**

## 前言

文件上传下载是个老生常谈的问题了，其中下载文件相对简单，后面一笔带过，本篇文章主要围绕着文件上传展开。

上传一个文件，在平常项目中，我们可以使用别人封装好的npm包如[multer](https://www.npmjs.com/package/multer)、[multiparty](https://www.npmjs.com/package/multiparty)；大型公司项目中，服务器存储不了很多的文件，成本也高，可以使用市面上已经很成熟的文件或对象存储服务，如阿里的[OSS](https://www.aliyun.com/product/oss)、[NAS](https://www.aliyun.com/product/nas)，腾讯的[COS](https://cloud.tencent.com/product/cos)、[CFS](https://cloud.tencent.com/product/cfs)，滴滴的[GIFT](https://base.xiaojukeji.com/console/gift#/home)，调用对应服务的api即可。

> 文件或对象存储的区别可参考这篇文章——[块存储、文件存储、对象存储这三者的本质差别是什么？](https://www.zhihu.com/question/21536660)

在这种大背景下，大家越来越忽略原理的实现。本篇文章舍弃所有包，带着大家从零到一实现一个简单的文件上传服务，包含切片上传、断点续传、文件秒传、展示上传进度条以及一些优化。

阅前热身：参考下图思考一下，你在什么段位 

![image-20220225003339608](https://tva1.sinaimg.cn/large/e6c9d24ely1gzp2lwxt13j20xm0jp40y.jpg)



## 前置知识点

### 文件上传为什么采用POST？

由于客户端上传的文件大小是不确定的，所以 HTTP 协议规定，文件上传的数据要存放于请求正文中，而不能出现在 URL 的地址栏中，因为地址栏中可以存放的数据量太小(一般不超过2K)

### 常见POST请求主体类型有哪些([Content-Type](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Content-Type))?

`application/x-www-form-urlencoded`: 数据被编码成以 `'&'` 分隔的键-值对, 同时以 `'='` 分隔键和值. 非字母或数字的字符会被 [percent-encoding](https://developer.mozilla.org/zh-CN/docs/Glossary/percent-encoding): 这也就是为什么这种类型不支持二进制数据(应使用 `multipart/form-data` 代替).

`multipart/form-data`：数据被编码成一条消息，以标签为单元，用分隔符分开。既可以上传键值对，也可以上传文件(原因：有boundary隔离。由于键值对方式，所以可以上传多个文件) 

`application/json` ：数据传输还是字符串，但会告诉服务端消息主体是序列化后的JSON字符串了解更多Content-Type类型，参考[菜鸟教程](https://www.runoob.com/http/http-content-type.html)

⚠️当 POST 请求是通过除 HTML 表单之外的方式发送时, 例如`XMLHttpRequest`, 那么请求主体可以是任何类型



## 先来一个登录功能练练手

>  [代码传送门](https://github.com/lemonliu2022/uploadDownload/tree/master/01form)

通过简单的登录功能，可以清楚的看到数据的传输格式，方便理解后面的数据处理

使用默认的 `application/x-www-form-urlencoded` 做为 content type 的请求体:

```json
Content-Length: 16
Content-Type: application/x-www-form-urlencoded
Host: localhost:3000
Origin: http://localhost:3000

user=foo&pwd=bar
```

使用 `multipart/form-data` 作为 content type 的请求体:

```json
Content-Length: 231
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryEmmEuLsfmB5sVvCw
Host: localhost:3000
Origin: http://localhost:3000

------WebKitFormBoundaryEmmEuLsfmB5sVvCw
Content-Disposition: form-data; name="user"

foo
------WebKitFormBoundaryEmmEuLsfmB5sVvCw
Content-Disposition: form-data; name="pwd"

bar
------WebKitFormBoundaryEmmEuLsfmB5sVvCw--
```

在后端使用「[bodyParser](https://www.npmjs.com/package/body-parser)」中间件接收这些数据，为了方便展示底层传输的信息，我手动监听了请求体[stream流](https://www.youtube.com/watch?time_continue=194&v=7_LPdttKXPc&feature=emb_logo)的传输，并打印出来，如下图。对比发现，跟前端请求的数据完全一致

![image-20220225003705309](https://tva1.sinaimg.cn/large/e6c9d24ely1gzp2ph7xhxj20t50femyn.jpg)

看明白了数据格式，灵活应用一下，将文本替换成文件，就可以实现一个简易的上传功能了 



## 曲线救国 - base64序列化数据上传

`application/x-www-form-urlencoded` 规定不能传输文件数据，我们可以将文件数据转为base64数据发送给后端，后端再解码转成Buffer存储成文件

> [代码传送门](https://github.com/lemonliu2022/uploadDownload/tree/master/02base64) 

![image-20220225003915363](https://tva1.sinaimg.cn/large/e6c9d24ely1gzp2rqjepfj20xv0tsjtq.jpg)

## FormData格式上传

和base64类似，服务器通过解析上传的文件数据并写入文件，较复杂的是文件数据的解析

[代码传送门](https://github.com/lemonliu2022/uploadDownload/tree/master/03formData)

我们试着上传一个文件看下传输的数据结构：

```json
------WebKitFormBoundaryL5AGcit70yhKB92Y
Content-Disposition: form-data; name="username"


lee
------WebKitFormBoundaryL5AGcit70yhKB92Y
Content-Disposition: form-data; name="password"


123456
Content-Disposition: form-data; name="file"; filename="upload.txt"
Content-Type: text/plain


upload
------WebKitFormBoundaryL5AGcit70yhKB92Y--
```

### 解析步骤

1. 获取分隔符。数据都被分隔符 `"------WebKitFormBoundaryL5AGcit70yhKB92Y"` 隔开，分隔符在每次上传都不同，但可以从 `req.headers['content-type']` 中获取，如：

```js
const contentType = req.headers['content-type']; // multipart/form-data; boundary=----WebKitFormBoundaryL5AGcit70yhKB92Y
const boundary = '--' + contentType.split('; ')[1].split('=')[1] // ------WebKitFormBoundaryL5AGcit70yhKB92Y
```

2. 提取数据。

   a. 标记数据的回车换行符

```json
------WebKitFormBoundaryL5AGcit70yhKB92Y\r\n
Content-Disposition: form-data; name="username"\r\n
\r\n
lee\r\n
------WebKitFormBoundaryL5AGcit70yhKB92Y\r\n
Content-Disposition: form-data; name="password"\r\n
\r\n
123456\r\n
Content-Disposition: form-data; name="file"; filename="upload.txt"\r\n
Content-Type: text/plain\r\n
\r\n
upload\r\n
------WebKitFormBoundaryL5AGcit70yhKB92Y--
```

​		b. 每段数据简化

```json
<分隔符>\r\n字段头\r\n\r\n内容\r\n
```

​		c. 用分隔符切分数据

```json
[
  '',
  "\r\n字段信息\r\n\r\n内容\r\n",
  "\r\n字段信息\r\n\r\n内容\r\n",
  "\r\n字段信息\r\n\r\n内容\r\n",
  '--'
]
```

​		d. 删除数据头尾数据

```json
[
  "\r\n字段信息\r\n\r\n内容\r\n",
  "\r\n字段信息\r\n\r\n内容\r\n",
  "\r\n字段信息\r\n\r\n内容\r\n",
]
```

​		f. 删除回车换行符

```json
[
	"字段信息", "内容",
	"字段信息", "内容",
	"字段信息", "内容",
]
```

3. Buffer 数据处理。

由于文件都是二进制数据，不能直接转换为字符串后再进行处理，否则数据会出错，所以我们需要node核心模块Buffer。如果对Buffer模块不熟悉的可以将它理解为数组

Buffer模块提供了indexOf方法获取Buffer数据中，其参数所在位置的index值

Buffer模块提供了slice方法，可通过index值切分Buffer数据

通过上面两个方法将数据通过 `/r/n` 分割并拼接起来——[代码传送门](https://github.com/lemonliu2022/uploadDownload/blob/master/03formData/utils/bufferSplit.js) 

![image-20220225004553256](https://tva1.sinaimg.cn/large/e6c9d24ely1gzp2ynb2nej20xv0isgnf.jpg)



## 后端使用第三方库——[multiparty](https://www.npmjs.com/package/multiparty)

虽然我们完成了完整的文件上传流程，但是实际工作中不可能从头开发所有功能，效率低而且通用型差。multiparty 包的实现和上面手写的差不多，项目中可以使用，具体API请参考[README](https://github.com/pillarjs/multiparty#readme)

> [代码传送门](https://github.com/lemonliu2022/uploadDownload/blob/master/03formData/app.js) 



## 大文件切片上传

> [代码传送门](https://github.com/lemonliu2022/uploadDownload/blob/master/04chunk)

如果太大的文件，比如一个视频1g 2g那么大，直接采用上面例子中的方法上传很可能会出链接现超时的情况，而且也会超过服务端允许上传文件的大小限制，解决这个问题我们可以将文件进行分片上传，每次只上传很小的一部分 比如2M。下面演示将文件切成3片的处理步骤

1. 把大文件切片化

file是Blob的实例，Blob.prototype.silce可以把一个文件切片处理

2. 同时并发三个切片的上传请求

​		/chunk 处理每一个切片的请求

​		chunk：文件切片

​		filename：切片的名字 ->文件名-索引.后缀

​		upload目录下创建一个以文件名命名的临时目录，把切片存储到这个目录下

​		xxxxxxxxxx-0-pn

​		gxxxxx-1-png

​		....

3. 等3个切片都上传完，再向服务器发送一个合并切片的请求

​		/merge 合并

​		filename：文件名 文件名.后缀

​		根据文件名找到之前存储的临时目录，按顺序读取目录中的切片信息，把每一个切片信息合并起来（合并完记得删除临时目录和里面的切片)

![image-20220225004919577](https://tva1.sinaimg.cn/large/e6c9d24ely1gzp327uv07j20xt0oydhq.jpg)



## 上传进度条

一个文件的上传进度条十分简单，使用ajax的 `upload.onprogress` 监听请求进度变化

如果是多个文件/切片乘以百分比即可



## 维持最大并发数

> [代码传送门](https://github.com/lemonliu2022/uploadDownload/blob/master/05requestAll)

在 Chrome 浏览器中**针对同一域名**允许的最大并发请求数目为 6，超过这一限制的后续请求将会被阻塞。 以下是 Chrome 浏览器关于最大请求链接数的一段介绍和相关[代码](https://chromium.googlesource.com/chromium/src/+/65.0.3325.162/net/socket/client_socket_pool_manager.cc#44)， Chrome 浏览器是不能修改这个值的，在源码里可以看到是固定写死的。

```json
https://chromium.googlesource.com/chromium/src/+/65.0.3325.162/net/socket/client_socket_pool_manager.cc#44
// Default to allow up to 6 connections per host. Experiment and tuning may
// try other values (greater than 0).  Too large may cause many problems, such
// as home routers blocking the connections!?!?  See http://crbug.com/12066.
int g_max_sockets_per_group[] = {
  6,  // NORMAL_SOCKET_POOL
  255 // WEBSOCKET_SOCKET_POOL
};
```

思考一下，如果文件过大，我们切片很多的情况，其他同域名请求就会被上传请求阻塞。我们需要实现一个类似浏览器的维持最大并发数的功能，让上传请求不要占满请求数

![image-20220225005041123](https://tva1.sinaimg.cn/large/e6c9d24ely1gzp33mkojhj20xp0ghabq.jpg)



## 文件秒传和断点续传

相同文件的文件名是可变的，内容是不变的，可以将文件/切片进行md5加密并存储在服务端/前端，下次上传相同文件/切片直接跳过

### 计算md5耗时问题

- 可以通过web-workder，还可以参考React的Fiber架构，通过requestIdleCallback来利用浏览器的空闲时间计算，不会卡死主线程

- 参考[布隆过滤器](https://zhuanlan.zhihu.com/p/43263751)的理念, 牺牲一点点的识别率来换取时间，比如我们可以抽样算hash



## 慢启动策略

[TCP拥塞控制的问题](https://link.juejin.cn/?target=https%3A%2F%2Fwww.zhihu.com%2Fquestion%2F58517416%2Fanswer%2F158142955) 其实就是根据当前网络情况，动态调整切片的大小



## 并发重试+报错

1. 请求出错.catch 把任务重新放在队列中

2. 出错后progress设置为-1 进度条显示红色

3. 数组存储每个文件hash请求的重试次数，做累加 比如 `[1, 0, 2]` 就是第0个文件切片报错1次，第2个报错2次

4. 超过3的直接 `reject`



## 文件碎片清理

如果很多人传了一半就离开，这些切片存在就没意义了，可以考虑定期清理，当然 ，我们可以使用[node-schedule](https://github.com/node-schedule/node-schedule)来管理定时任务 比如我们每天扫一次target，如果文件的修改时间是一个月以前了，就直接删除吧



## 文件下载

> [代码传送门](https://github.com/lemonliu2022/uploadDownload/blob/master/06download)

### 直接下载

- 浏览器地址栏输入文件URL

- window.location.href = URL

- window.open(URL)

- a 标签 download 属性

### 直接下载(后端兼容处理 attachment)

浏览器识别的文件默认进行了解析，我们可以声明文件的 header 的 Content-Dispositon 信息，告诉浏览器该链接为下载附件链接

```json
Content-Disposition: attachment; filename="filename.xls" 
```

