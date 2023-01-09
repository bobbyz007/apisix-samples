1. 激活 response-rewrite 插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - proxy-rewrite
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
        "proxy-rewrite": {
            "uri": "/get?c=d",
            "host": "172.18.0.1:1980",
            "headers": {
               "set": {
                    "X-Api-Version": "v1",
                    "X-Api-Engine": "apisix",
                    "X-Api-useless": ""
                },
                "add": {
                    "X-Request-ID": "112233"
                },
                "remove":[
                    "X-test"
                ]
            }
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

3. 访问验证： 发出批量请求
```shell
curl -X GET http://127.0.0.1:9080/anything/test?a=b
```
返回结果：proxy已经将uri改写为/get?c=d
```shell
{
  "args": {
    "a": "b", 
    "c": "d"
  }, 
  "headers": {
    "Accept": "*/*", 
    "Host": "172.18.0.1:1980", 
    "User-Agent": "curl/7.68.0", 
    "X-Api-Engine": "apisix", 
    "X-Api-Version": "v1", 
    "X-Forwarded-Host": "127.0.0.1"
  }, 
  "origin": "172.18.0.1", 
  "url": "http://127.0.0.1/get?c=d&a=b"
}
```