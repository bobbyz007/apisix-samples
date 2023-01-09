1. 激活插件：batch-request, public-api
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - batch-requests
  - public-api
```
配置修改记得重启apisix

2. 创建route: anything，被batch-requests批量调用
```shell
curl "http://127.0.0.1:9180/apisix/admin/routes/1" -H "X-API-KEY: edd1c9f034335f136f87ad84b625c8f1" -X PUT -d '
{
  "uri": "/anything/*",
  "name": "anything",
  "methods": ["GET"],
  "upstream": {
    "type": "roundrobin",
    "nodes": {
      "172.18.0.1:1980": 1
    }
  }
}'
```

3创建route，暴露endpoint： 
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/br -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/apisix/batch-requests",
    "plugins": {
        "public-api": {}
    }
}'
```

4. 访问验证： 发出批量请求
```shell
curl --location --request POST 'http://127.0.0.1:9080/apisix/batch-requests' --header 'Content-Type: application/json' --data '{
    "headers": {
        "Content-Type": "application/json",
        "admin-jwt":"xxxx"
    },
    "timeout": 500,
    "pipeline": [
        {
            "method": "GET",
            "path": "/anything/test?a=b"
        },
        {
            "method": "GET",
            "path": "/anything/test?c=d"
        }
    ]
}'
```
返回结果如下：
```shell
[
  {
    "body": "{\n  \"args\": {\n    \"a\": \"b\"\n  }, \n  \"data\": \"\", \n  \"files\": {}, \n  \"form\": {}, \n  \"headers\": {\n    \"Accept\": \"*/*\", \n    \"Admin-Jwt\": \"xxxx\", \n    \"Content-Type\": \"application/json\", \n    \"Host\": \"127.0.0.1:9080\", \n    \"User-Agent\": \"curl/7.68.0\", \n    \"X-Forwarded-Host\": \"127.0.0.1\"\n  }, \n  \"json\": null, \n  \"method\": \"GET\", \n  \"origin\": \"172.18.0.1\", \n  \"url\": \"http://127.0.0.1/anything/test?a\u003db\"\n}\n",
    "reason": "OK",
    "headers": {
      "Date": "Mon, 09 Jan 2023 07:07:08 GMT",
      "Server": "APISIX/3.1.0",
      "Access-Control-Allow-Credentials": "true",
      "Connection": "keep-alive",
      "Content-Length": "402",
      "Access-Control-Allow-Origin": "*",
      "Content-Type": "application/json"
    },
    "status": 200
  },
  {
    "body": "{\n  \"args\": {\n    \"c\": \"d\"\n  }, \n  \"data\": \"\", \n  \"files\": {}, \n  \"form\": {}, \n  \"headers\": {\n    \"Accept\": \"*/*\", \n    \"Admin-Jwt\": \"xxxx\", \n    \"Content-Type\": \"application/json\", \n    \"Host\": \"127.0.0.1:9080\", \n    \"User-Agent\": \"curl/7.68.0\", \n    \"X-Forwarded-Host\": \"127.0.0.1\"\n  }, \n  \"json\": null, \n  \"method\": \"GET\", \n  \"origin\": \"172.18.0.1\", \n  \"url\": \"http://127.0.0.1/anything/test?c\u003dd\"\n}\n",
    "reason": "OK",
    "headers": {
      "Date": "Mon, 09 Jan 2023 07:07:08 GMT",
      "Server": "APISIX/3.1.0",
      "Access-Control-Allow-Credentials": "true",
      "Connection": "keep-alive",
      "Content-Length": "402",
      "Access-Control-Allow-Origin": "*",
      "Content-Type": "application/json"
    },
    "status": 200
  }
]
```