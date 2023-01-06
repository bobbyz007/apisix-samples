1. 创建consumer
```shell
curl http://127.0.0.1:9180/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "username": "jack",
    "plugins": {
        "jwt-auth": {
            "key": "user-key",
            "secret": "my-secret-key"
        }
    }
}'
```

2. 创建route
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/anything/*",
    "plugins": {
        "jwt-auth": {}
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "172.18.0.1:1980": 1
        }
    }
}'
```

3. 用public-api插件暴露服务endpoint: /apisix/plugin/jwt/sign
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/jas -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/apisix/plugin/jwt/sign",
    "plugins": {
        "public-api": {}
    }
}'
```

4. 根据暴露的endpoint使用consumer中配置的 user-key 生成jwt token
```shell
curl http://127.0.0.1:9080/apisix/plugin/jwt/sign?key=user-key -i

HTTP/1.1 200 OK
Date: Fri, 06 Jan 2023 08:11:15 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.1.0

`eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NzMwNzkwNzUsImtleSI6InVzZXIta2V5In0.pLOnBhK_4U7TOuAhtIPoQXrY3SxesEpb6l6d-DuRtKs`
```
也可以添加payload：
```shell
curl -G --data-urlencode 'payload={"uid":10000,"uname":"test"}' http://127.0.0.1:9080/apisix/plugin/jwt/sign?key=user-key -i
```

5. 验证访问: 访问时带上生成的token
```shell
# 成功访问
curl http://127.0.0.1:9080/anything/test?a=b -H 'Authorization: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NzMwNzkwNzUsImtleSI6InVzZXIta2V5In0.pLOnBhK_4U7TOuAhtIPoQXrY3SxesEpb6l6d-DuRtKs' -i
curl http://127.0.0.1:9080/anything/test?a=b --cookie jwt=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJleHAiOjE2NzMwNzkwNzUsImtleSI6InVzZXIta2V5In0.pLOnBhK_4U7TOuAhtIPoQXrY3SxesEpb6l6d-DuRtKs -i


# 访问失败
curl http://127.0.0.1:9080/anything/test?a=b -i
```