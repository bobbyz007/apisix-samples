1. 激活 mocking 插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - mocking
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 创建route
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/anything/*",
    "plugins": {
        "mocking": {
            "delay": 1,
            "content_type": "application/json",
            "response_status": 200,
            "response_schema": {
               "properties":{
                   "field0":{
                       "example":"abcd",
                       "type":"string"
                   },
                   "field1":{
                       "example":123.12,
                       "type":"number"
                   },
                   "field3":{
                       "properties":{
                           "field3_1":{
                               "type":"string"
                           },
                           "field3_2":{
                               "properties":{
                                   "field3_2_1":{
                                       "example":true,
                                       "type":"boolean"
                                   },
                                   "field3_2_2":{
                                       "items":{
                                           "example":155.55,
                                           "type":"integer"
                                       },
                                       "type":"array"
                                   }
                               },
                               "type":"object"
                           }
                       },
                       "type":"object"
                   },
                   "field2":{
                       "items":{
                           "type":"string"
                       },
                       "type":"array"
                   }
               },
               "type":"object"
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

3. 访问验证
```shell
curl "http://127.0.0.1:9080/anything/test?a=b" -i
```
返回结果：延迟1s返回mock结果
```shell
HTTP/1.1 200 OK
Date: Mon, 09 Jan 2023 13:06:38 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
x-mock-by: APISIX/3.1.0
Server: APISIX/3.1.0

{
  "field0": "abcd",
  "field1": 123.12,
  "field2": [
    "b",
    "th"
  ],
  "field3": {
    "field3_1": "oi",
    "field3_2": {
      "field3_2_1": true,
      "field3_2_2": [
        155,
        155,
        155
      ]
    }
  }
}
```

如果mocking设置response_example:
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/anything/*",
    "plugins": {
        "mocking": {
            "delay":0,
            "content_type":"application/json",
            "with_mock_header":true,
            "response_status":201,
            "response_example":"{\"a\":1,\"b\":2}"
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
返回结果：返回response_example设置的内容
```shell
HTTP/1.1 201 Created
Date: Mon, 09 Jan 2023 13:13:33 GMT
Content-Type: application/json
Transfer-Encoding: chunked
Connection: keep-alive
x-mock-by: APISIX/3.1.0
Server: APISIX/3.1.0

{"a":1,"b":2}
```