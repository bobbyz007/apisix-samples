1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - ua-restriction
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
        "ua-restriction": {
             "bypass_missing": true,
             "allowlist": [
                 "my-bot1",
                 "(Baiduspider)/(\\d+)\\.(\\d+)"
             ],
             "denylist": [
                 "my-bot2",
                 "(Twitterspider)/(\\d+)\\.(\\d+)"
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

# 添加denylist的user agent，则拒绝访问
curl http://127.0.0.1:9080/anything/test?a=b -i -H 'User-Agent: Twitterspider/2.0'

HTTP/1.1 403 Forbidden
Date: Tue, 10 Jan 2023 04:56:47 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.1.0

{"message":"Do you want to do something bad?"}
```