echo仅用于测试，不要用于生产环境。

1. 激活echo插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - echo
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 创建route
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
        "echo": {
            "before_body": "before the body modification "
        }
    },
    "upstream": {
        "nodes": {
            "172.18.0.1:1980": 1
        },
        "type": "roundrobin"
    },
    "uri": "/anything/*"
}'
```

3. 访问验证： 发出批量请求
```shell
curl -i http://127.0.0.1:9080/anything/test?a=b
```
返回结果显示拦截了before_body:
```shell
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
Date: Mon, 09 Jan 2023 07:37:31 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.1.0

before the body modification {
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