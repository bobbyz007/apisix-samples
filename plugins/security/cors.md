1. 激活 cors 插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - cors
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 创建route
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/anything/*",
    "plugins": {
        "cors": {}
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
curl http://127.0.0.1:9080/anything/test?a=b -i
```
返回结果：返回了CORS headers: Access-Control-Allow-Methods等
```shell
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 335
Connection: keep-alive
Date: Tue, 10 Jan 2023 03:06:59 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.1.0
Access-Control-Allow-Methods: *
Access-Control-Max-Age: 5
Access-Control-Expose-Headers: *
Access-Control-Allow-Headers: *

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