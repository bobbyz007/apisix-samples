1. 创建consumer
```shell
curl http://127.0.0.1:9180/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "username": "jack",
    "plugins": {
        "key-auth": {
            "key": "auth-one"
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
    "id": 1,
    "plugins": {
        "key-auth": {}
    },
    "upstream": {
        "type": "roundrobin",
        "nodes": {
            "172.18.0.1:1980": 1
        }
    }
}'
```

3. 访问验证：
```shell
# 访问认证成功
curl http://127.0.0.1:9080/anything/test?a=b -H 'apikey: auth-one' -i
# 访问认证失败
curl http://127.0.0.1:9080/anything/test?a=b -H 'apikey: auth-xxx' -i
curl http://127.0.0.1:9080/anything/test?a=b -i
```