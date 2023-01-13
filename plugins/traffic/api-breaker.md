作用： 实现了 API 熔断功能，从而帮助我们保护上游业务服务。

1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - api-breaker
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

3. 在route 上配置
```shell
curl "http://127.0.0.1:9180/apisix/admin/routes/1" \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
        "api-breaker": {
            "break_response_code": 505,
            "break_response_body": "upstream is unhealthy",
            "unhealthy": {
                "http_statuses": [500, 501, 502, 503, 504],
                "failures": 3
            },
            "healthy": {
                "http_statuses": [200],
                "successes": 2
            }
        }
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "172.18.0.1:1980": 1
        }
    },
    "uri": "/status/*"
}'
```

3. 访问验证
利用httpbin的status/[code] 服务来验证：
```shell
# 访问3次 返回正常500
curl -i http://127.0.0.1:9080/status/500

HTTP/1.1 500 INTERNAL SERVER ERROR
Content-Type: text/html; charset=utf-8
Content-Length: 0
Connection: keep-alive
Date: Thu, 12 Jan 2023 09:05:05 GMT
Access-Control-Allow-Origin: *
Access-Control-Allow-Credentials: true
Server: APISIX/3.1.0
X-APISIX-Upstream-Status: 500

# 访问第4次开始 返回熔断的505
curl -i http://127.0.0.1:9080/status/500

HTTP/1.1 505 HTTP Version Not Supported
Date: Thu, 12 Jan 2023 09:07:46 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.1.0

upstream is unhealthy
```
