# 前端配置

`/config/dev.env.js`中配置BASE_API

```js
module.exports = merge(prodEnv, {
  NODE_ENV: '"development"',
  BASE_API: '"http://localhost:9001"'
})
```

之后请求的时候，获取BASE_API的方法：

```js
BASE_API: process.env.BASE_API // 接口API地址
```

# nginx配置

nginx的功能有以下几种：

- 请求转发
- 负载均衡
- 动静分离

我们做请求转发需要配置的文件，在`conf/nginx.conf`目录下

```js
    server {
        listen 9001;
        server_name localhost;
        location ~/eduService/ {
            proxy_pass http://localhost:8001;
        }
        location ~/eduOss/ {
            proxy_pass http://localhost:8002;
        }
        location ~/eduVod/ {
            proxy_pass http://localhost:8003;
        }
    }
```

nginx启动命令：

```shell
nginx  启动
nginx -s stop  停止
```

# 后端配置

后端的话，每个服务对应不同的路由，配置不同的`server.port`即可。

# 问题 Request Entity Too Large

`Access to XMLHttpRequest at 'http://localhost:9001/eduVod/video/uploadAliyun' from origin 'http://localhost:9528' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.`

跨域问题产生的原因，同源策略。

后端在controller上加@CrossOrigin注解可以解决。

但是这一次是视频上传，因为使用nginx做请求分发，nginx有上传文件大小的限制需要配置

![image-20200729153746086](C:\Users\13327\AppData\Roaming\Typora\typora-user-images\image-20200729153746086.png)

```js
http {
    include       mime.types;
    default_type  application/octet-stream;
    client_max_body_size 1024m;# 设置大小
    server {
        listen 9001;
        server_name localhost;
        location ~/eduService/ {
            proxy_pass http://localhost:8001;
        }
        location ~/eduOss/ {
            proxy_pass http://localhost:8002;
        }
        location ~/eduVod/ {
            proxy_pass http://localhost:8003;
        }
    }

}
```

