# 抖音APP数据接口加密算法分析
﻿抖音作为一款日活超过1亿的优秀APP，其客户端与服务端的通信方式很值得APP开发者去研究和学习，为了保护其数据，客户端请求数据的接口都进行了加密，未经过加密处理的url，请求的时候不会返回数据，这里以最新的3.0版本为例，分析一下加密算法。  

>https://api.amemv.com/aweme/v1/feed/?manifest_version_code=310&_rticket=1541061729354&pull_type=2&app_type=normal&iid=48910484145&channel=aweGW&device_type=Redmi+5A&language=zh&type=0&uuid=868661038685020&resolution=720*1280&openudid=7664f169100e4e94&update_version_code=3102&os_api=25&max_cursor=0&filter_warn=0&need_relieve_aweme=0&dpi=320&ac=wifi&device_id=58329658832&os_version=7.1.2&count=6&version_code=310&is_cold_start=0&volume=0.0&app_name=aweme&req_from=&version_name=3.1.0&js_sdk_version=&device_brand=Xiaomi&ssmix=a&device_platform=android&min_cursor=-1&aid=1128&ts=1541061729&as=a1e58b7dd1960b1c1a4355&cp=bc64b9541ca0d2c8e1cDgM&mas=01b9fcc9ce21a0deb29046deb46e30350aacaccc2c868cc68c460c
>

上面的url用于抖音首页推荐列表数据的请求，可以看到除了一些设备、网络、渠道等相关信息以外，url尾部有as、cp和mas三个参数，这三个参数用于加密url，它们是根据前面的参数计算出来的。我们先看一下Java部分：![Java部分1](https://img-blog.csdnimg.cn/20181101170129884.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdteGU=,size_16,color_FFFFFF,t_70)
大概的计算流程如下：

 1. 把url中的参数提取出来，按顺序存到一个List中；
 2. 获取device_id；
 3. 调用`UserInfo.getUserInfo(ts, (String[]) list.toArray(new String[list.size()]), null, device_id)`获取一个44位加密后的字符串， 第1个参数是时间戳，第2个参数是第1步的list转换成的数组，第4个参数是第2部获取的device_id；这个方法是在native中实现；
 4. 把第3部计算的字符串前22位赋值给as，后22位赋值给cp;
 5. 调用`Lcom/ss/sys/ces/f/a;->a(as.getBytes())[B`获取一个27位的byte数组，最终会调到`Lcom/ss/sys/ces/a;->e([B)[B`，这个方法也是native实现；
 6. 把第5部计算的27位byte数组传入`Lcom/ss/android/common/applog/k;->a([B)Ljava/lang/String;`，结果为54位的字符串，赋值给mas，k.a()的实现如下图，走最后一个else分支； 
 ![java部分2](https://img-blog.csdnimg.cn/20181101173341958.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dhbmdteGU=,size_16,color_FFFFFF,t_70)  
   
可以看到抖音url加密算法的核心部分采用c/c++语言来实现，对native部分的逆向分析比java部分难度更大，而且抖音增加了反调试技术并且对native函数的名字进行了混淆，对代码逻辑进行ollvm混淆，进一步增加了分析的难度；通过分析，我们发现抖音的加密算法在libcms.so中，利用IDA静态分析加上[inline hook](https://github.com/ele7enxxh/Android-Inline-Hook)技术，我们找到了getUserInfo的native实现函数sub_26750(0x26750是函数的相对so文件起始位置的偏移):
![native 1](https://img-blog.csdnimg.cn/20181101180639160.png)
`Lcom/ss/sys/ces/a;->e([B)[B`的native实现函数是sub_2f1c8()
![native2](https://img-blog.csdnimg.cn/20181101182626410.png)

 native部分的具体细节这里不在列出，大家可以参考看雪论坛上的两篇文章：
 
 1. [\[原创\]抖音接口加密签名协议](https://bbs.pediy.com/thread-226931.htm)
 2. [抖音签名算法的逆向杂谈](https://zhuanlan.kanxue.com/article-5010.htm)
 
最后，对抖音API加密算法感兴趣的朋友，可以私下[联系我](http://47.105.95.219)。
