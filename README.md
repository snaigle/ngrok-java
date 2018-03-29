初始代码来源于@wendal 的项目，以下是项目地址
==================================
https://github.com/nutzam/nutzmore/tree/master/nutz-plugins-ngrok

为什么要开一个新项目
==================================
原项目放在nutzmore里，不方便fork


简介
==================================

用Java实现的Ngrok的服务器端和客户端.

什么是Ngrok
==================================

ngrok是一个内网穿透隧道, 可以将你的web服务映射到外网的服务器上,从而可以被其他人访问到.


功能进度
==================================

- [x] 客户端端
- [x] 服务器端
- [x] 服务器端Redis鉴权
- [x] 基于域名的流量统计
- [ ] 客户端的拦截器机制
- [ ] 客户端web界面

服务器端和客户端均兼容Ngrok原版.

客户端用法
===================================

### 命令行方式

```
java -cp nutz-plugins-ngrok.jar org.nutz.plugins.ngrok.client.NgrokClient \ 
    -srv_host=xxxx.cn -srv_port=4443 \
    -to_host=127.0.0.1 -to_port=8080 \
    -auth_token=your_token
    -conf_file=xxxx.properties 

```
conf_file中的key 就是 srv_host=xxxx.cn 这样，去掉前面的`-`即可  

conf_file为空时默认会加载 ~/.ngork-java/config.properties 

命令行中的参数会覆盖掉conf_file中的key

   
服务器端用法
=================================

服务器端部署需要什么:

* JDK 8最新版
* 若使用redis鉴权, 那么需要redis
* https证书, 合法的,不是自签名的
* 服务器必须有一个外网ip

使用redis与否的区别:

* 非redis模式下,域名是随机6位字母,填写任意access token即可.
* redis模式下, 会执行hget ngrok token 获取域名前缀, 不再是随机域名.

### 如果你没有https证书??

请访问: https://github.com/diafygi/acme-tiny

按页面上的步骤, 生成crt和key文件

### 然后用https证书生成jks文件

使用 crt和key文件, 也就是nginx使用的证书,生成jks的步骤

### 首先, 使用openssl生成p12文件,必须输入密码

```
openssl pkcs12 -export -in 1_wendal.cn_bundle.crt -inkey 2_wendal.cn.key -out wendal.cn.p12
```

### 再然后, 使用keytool 生成jks

```
keytool -importkeystore -destkeystore wendal.cn.jks -srckeystore wendal.cn.p12 -srcstoretype pkcs12 -alias 1
```

目标文件是 wendal.cn.jks 即     域名.jks

### 最后,启动NgrokServer,普通模式

跟普通java程序一下, 传点参数,启动它就可以了

```
java -cp nutz-plugins-ngrok.jar org.nutz.plugins.ngrok.server.NgrokServer -srv_host=wendal.cn -ssl_jks_path=wendal.cn.jks
```

启动日志:

```
2017-3-16 21:59:44.882 INFO [main] Select SystemLog as Nutz.Log implement
2017-3-16 21:59:44.922 DEBUG [main] config key=redis value=true
2017-3-16 21:59:44.975 DEBUG [main] Using 95 castor for Castors
2017-3-16 21:59:44.977 DEBUG [main] NgrokServer start ...
2017-3-16 21:59:44.977 DEBUG [main] try to load Java KeyStore File ...
2017-3-16 21:59:44.985 DEBUG [main] load jks from wendal.cn.jks
2017-3-16 21:59:45.488 DEBUG [main] using default CachedThreadPool
2017-3-16 21:59:45.491 DEBUG [main] using default auth provider
2017-3-16 21:59:45.571 DEBUG [main] start listen srv_port=4443
2017-3-16 21:59:45.574 DEBUG [main] start listen http_port=9080
2017-3-16 21:59:45.575 DEBUG [main] start Contrl Thread...
2017-3-16 21:59:45.575 DEBUG [main] start Http Thread...
```

### 以Redis模式启动NgrokServer

```
java -cp nutz-plugins-ngrok.jar org.nutz.plugins.ngrok.server.NgrokServer -srv_host=wendal.cn -ssl_jks_path=wendal.cn.jks -redis=true
```

```
2017-3-16 21:59:44.882 INFO [main] Select SystemLog as Nutz.Log implement
2017-3-16 21:59:44.922 DEBUG [main] config key=redis value=true
2017-3-16 21:59:44.975 DEBUG [main] Using 95 castor for Castors
2017-3-16 21:59:44.977 DEBUG [main] NgrokServer start ...
2017-3-16 21:59:44.977 DEBUG [main] try to load Java KeyStore File ...
2017-3-16 21:59:44.985 DEBUG [main] load jks from wendal.cn.jks
2017-3-16 21:59:45.488 DEBUG [main] using default CachedThreadPool
2017-3-16 21:59:45.491 DEBUG [main] using default auth provider
2017-3-16 21:59:45.571 DEBUG [main] start listen srv_port=4443
2017-3-16 21:59:45.574 DEBUG [main] start listen http_port=9080
2017-3-16 21:59:45.575 DEBUG [main] start Contrl Thread...
2017-3-16 21:59:45.575 DEBUG [main] start Http Thread...
```

请在redis下执行

```
hset ngrok aabbccddeeff hello,hi
```

使用 access token=aabbccddeeff 进行登录, 即可拥有 hello.wendal.cn 和 hi.wendal.cn 两个隧道域名

### Nginx配置实例

默认情况下, NgrokServer监听的是9080端口, 而通常访问的端口是80, 所以有两个选择

* 让NgrokServer监听80端口,但其他程序就没法使用80端口了
* 让nginx/apache/iis监听80端口, 按域名规则转发给NgrokServer

因为我最熟的就是nginx,下面给出nginx的配置实例:

值得注意的几个点:

* 不要使用upstream, 不要使用proxy_http_version 1.1; 会导致请求乱转发
* 关闭gzip,否则etag会被nginx过滤掉,使得资源文件没法通过etag相等来判断是否发送304响应,会消耗一些流量.

```
        server {
            listen 80;
            server_name www.wendal.cn wendal.cn;
            location / {
                    proxy_pass http://127.0.0.1:8181;
                    proxy_set_header Host $http_host;
                    proxy_set_header X-Forwarded-For $remote_addr;
                    proxy_send_timeout 1h;
                    proxy_read_timeout 1h;
                    client_max_body_size 128m;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection $http_connection;
            }
        }

        server {
            listen 80;
            server_name *.ngrok.wendal.cn;
            gzip off;
            location / {
                    proxy_pass http://127.0.0.1:9080;
                    proxy_set_header Host $http_host;
                    proxy_set_header X-Forwarded-For $remote_addr;
                    proxy_send_timeout 1h;
                    proxy_read_timeout 1h;
                    client_max_body_size 128m;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection $http_connection;
            }
        }

        server {
            listen 80;
            server_name "~^[a-z0-9]{6}.wendal.cn$";
            gzip off;
            resolver 8.8.8.8;
            location / {
                    proxy_pass http://127.0.0.1:9080;
                    proxy_set_header Host $http_host;
                    proxy_set_header X-Forwarded-For $remote_addr;
                    proxy_send_timeout 1h;
                    proxy_read_timeout 1h;
                    client_max_body_size 128m;
                    proxy_set_header Upgrade $http_upgrade;
                    proxy_set_header Connection $http_connection;

            }
        }
```