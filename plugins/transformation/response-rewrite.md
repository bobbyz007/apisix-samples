1. 激活 response-rewrite 插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - response-rewrite
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 创建route
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/anything/*",
    "plugins": {
        "response-rewrite": {
            "body": "{\"code\":\"ok\",\"message\":\"new json body\"}",
            "headers": {
                "set": {
                    "X-Server-id": 3,
                    "X-Server-status": "on",
                    "X-Server-balancer_addr": "$balancer_ip:$balancer_port"
                }
            },
            "vars":[
                [ "status","==",200 ]
            ]
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
curl -X GET -i  http://127.0.0.1:9080/anything/test?a=b
```
返回结果：header和body已经改写
```shell
HTTP/1.1 200 OK
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
Date: Mon, 09 Jan 2023 08:24:06 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.1.0
X-Server-id: 3
X-Server-status: on
X-Server-balancer-addr: 172.18.0.1:1980

{"code":"ok","message":"new json body"}
```