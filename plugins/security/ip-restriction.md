1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - ip-restriction
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
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "172.18.0.1:1980": 1
        }
    },
    "plugins": {
        "ip-restriction": {
            "whitelist": [
                "127.0.0.1",
                "172.18.0.1",
                "113.74.26.106/24"
            ],
            "message": "Do you want to do something bad?"
        }
    }
}'
```

3. 访问验证
```shell
# 在白名单正常访问
curl http://127.0.0.1:9080/anything/test?a=b -i

# 在whitelist删除 172.18.0.0.1，则会拒绝访问
HTTP/1.1 403 Forbidden
Date: Tue, 10 Jan 2023 03:26:41 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.1.0

{"message":"Do you want to do something bad?"}
```