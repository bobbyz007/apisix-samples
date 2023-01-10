1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - uri-blocker
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 创建route
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/*",
    "plugins": {
        "uri-blocker": {
            "block_rules": ["root.exe", "root.m+"],
            "rejected_msg": "access is not allowed."
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
curl -i http://127.0.0.1:9080/root.exe?a=a
```
返回结果：
```shell
HTTP/1.1 403 Forbidden
Date: Tue, 10 Jan 2023 03:17:38 GMT
Content-Type: application/octet-stream
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.1.0

{"error_msg":"access is not allowed."}
```