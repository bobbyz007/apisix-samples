作用： 为每一个请求代理添加 unique ID 用于追踪 API 请求。

1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - request-id
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

3. 在route 上配置

将客户端的 IP 地址作为限制请求速率的条件
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/anything/*",
    "plugins": {
        "request-id": {
            "include_in_response": true
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

3. 访问验证
```shell
# 每次请求响应头返回不同的 X-Request-Id
curl -i http://127.0.0.1:9080/anything/test?a=b

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 335
Connection: keep-alive
Date: Fri, 13 Jan 2023 07:36:17 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.1.0
X-Request-Id: 51dd1d53-2c7e-445e-bf57-a7f206ade752

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 335
Connection: keep-alive
Date: Fri, 13 Jan 2023 07:37:08 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.1.0
X-Request-Id: 3695e151-0e55-40f1-95d1-f5a60aac5b04

```
