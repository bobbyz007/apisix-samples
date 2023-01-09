1. 激活 fault-injection 插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - fault-injection
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 创建route：请求延迟3s，uri参数name为jack时直接返回注入的fault错误
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
        "fault-injection": {
            "abort": {
                    "http_status": 403,
                    "body": "Fault Injection!\n",
                    "vars": [
                        [
                            [ "arg_name","==","jack" ]
                        ]
                    ]
            },
            "delay": {
              "duration": 3
           }
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

3. 访问验证
```shell
curl "http://127.0.0.1:9080/anything/test?name=allen&a=b" -i
```
返回结果：延迟3s返回正常结果
```shell
HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 368
Connection: keep-alive
Date: Mon, 09 Jan 2023 12:45:58 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.1.0

{
  "args": {
    "a": "b", 
    "name": "allen"
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
  "url": "http://127.0.0.1/anything/test?name=allen&a=b"
}
```
```shell
curl "http://127.0.0.1:9080/anything/test?name=jack&a=b" -i
```
返回结果：延迟3s，同时匹配abort条件返回注入错误
```shell
HTTP/1.1 403 Forbidden
Date: Mon, 09 Jan 2023 12:46:31 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.1.0

Fault Injection!
```