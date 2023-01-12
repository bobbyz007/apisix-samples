作用： 基于固定的时间窗口，限制单个客户端在指定的时间范围内对服务的总请求数

1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - limit-count
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 在route 上配置

将客户端的 IP 地址作为限制请求速率的条件:在60s内的时间窗口内，请求数阈值为2
```shell
curl -i http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/anything/*",
    "plugins": {
        "limit-count": {
            "count": 2,
            "time_window": 60,
            "rejected_code": 503,
            "key_type": "var",
            "key": "remote_addr"
        }
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "172.18.0.1:1980": 1
        }
    }
}'
```

访问验证：
```shell
# 60s 内访问前2次成功，第3次 开始失败
curl -i http://127.0.0.1:9080/anything/test?a=b

# 成功访问时会返回响应头X-RateLimit-*
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 335
Connection: keep-alive
X-RateLimit-Limit: 2
X-RateLimit-Remaining: 1
Date: Thu, 12 Jan 2023 05:20:03 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.1.0

{
  "args": {
    "a": "b"
  }, 
  "data": "", 
  "files": {}, 
  "form": {}, 
  "headers": {
    "Accept": "*/*", 
    "Host": "127.0.0.1:9080", 
    "User-Agent": "curl/7.68.0", 
    "X-Forwarded-Host": "127.0.0.1"
  }, 
  "json": null, 
  "method": "GET", 
  "origin": "172.18.0.1", 
  "url": "http://127.0.0.1/anything/test?a=b"
}

```

3. 多个路由共享同一个限流计数器
创建一个服务：
```shell
curl -i http://127.0.0.1:9180/apisix/admin/services/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
        "limit-count": {
            "count": 2,
            "time_window": 60,
            "rejected_code": 503,
            "key": "remote_addr",
            "group": "services_1#1640140621"
        }
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "172.18.0.1:1980": 1
        }
    }
}'
```
不同路由绑定同一个服务：
```shell
curl -i http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "service_id": "1",
    "uri": "/anything/*"
}'
```
```shell
curl -i http://127.0.0.1:9180/apisix/admin/routes/2 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "service_id": "1",
    "uri": "/get"
}'
```

这样路由1和2共享同一个限流计数器:在60s内的时间窗口内，请求数阈值为2
```shell
# 第一次：正常访问
curl -i http://127.0.0.1:9080/anything/test?a=b

# 第二次：正常访问
curl -i http://127.0.0.1:9080/get?c=d

# 第三次开始：访问拒绝，无论是请求anything/test?a=b 或 get?c=d
```
