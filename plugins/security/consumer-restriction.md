1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - consumer-restriction
  - basic-auth
  - key-auth
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 根据consumer name来限制
```shell
curl http://127.0.0.1:9180/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -i -d '
{
    "username": "jack1",
    "plugins": {
        "basic-auth": {
            "username":"jack2019",
            "password": "123456"
        }
    }
}'
```
```shell
curl http://127.0.0.1:9180/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -i -d '
{
    "username": "jack2",
    "plugins": {
        "basic-auth": {
            "username":"jack2020",
            "password": "123456"
        }
    }
}'
```
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
        "basic-auth": {},
        "consumer-restriction": {
            "whitelist": [
                "jack1"
            ]
        }
    }
}'
```
consumer name验证：
```shell
# 在白名单正常访问
curl -u jack2019:123456 http://127.0.0.1:9080/anything/test?a=b -i

# 不在白名单中，拒绝访问
curl -u jack2020:123456 http://127.0.0.1:9080/anything/test?a=b -i

HTTP/1.1 403 Forbidden
Date: Tue, 10 Jan 2023 06:07:55 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.1.0

{"message":"The consumer_name is forbidden."}
```

3. 根据某个consumer的允许的method来限制
```shell
curl http://127.0.0.1:9180/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -i -d '
{
    "username": "jack1",
    "plugins": {
        "basic-auth": {
            "username":"jack2019",
            "password": "123456"
        }
    }
}'
```
```shell
curl http://127.0.0.1:9180/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -i -d '
{
    "username": "jack2",
    "plugins": {
        "basic-auth": {
            "username":"jack2020",
            "password": "123456"
        }
    }
}'
```
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
        "basic-auth": {},
        "consumer-restriction": {
            "allowed_by_methods":[{
                "user": "jack1",
                "methods": ["POST"]
            }]
        }
    }
}'
```
根据某个consumer允许的method验证：
```shell
# jack2019的get请求被拒绝，只允许post请求
curl -u jack2019:123456 http://127.0.0.1:9080/anything/test?a=b -i
```

4. 根据service_id来限制：
```shell
curl http://127.0.0.1:9180/apisix/admin/services/1 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "upstream": {
        "nodes": {
            "172.18.0.1:1980": 1
        },
        "type": "roundrobin"
    },
    "desc": "new service 001"
}'
```
```shell
curl http://127.0.0.1:9180/apisix/admin/services/2 -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "upstream": {
        "nodes": {
            "172.18.0.1:1980": 1
        },
        "type": "roundrobin"
    },
    "desc": "new service 002"
}'
```
创建consumer： 配置 consumer-restriction 插件，允许访问id为1的service
```shell
curl http://127.0.0.1:9180/apisix/admin/consumers -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "username": "new_consumer",
    "plugins": {
    "key-auth": {
        "key": "auth-jack"
    },
    "consumer-restriction": {
           "type": "service_id",
            "whitelist": [
                "1"
            ],
            "rejected_code": 403
        }
    }
}'
```
绑定route到service_id为1：
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
    "service_id": 1,
    "plugins": {
         "key-auth": {
        }
    }
}'
```

根据service_id验证：
```shell
# 正常访问，因为consumer白名单的service_id是1， route绑定的service_id也是1
curl http://127.0.0.1:9080/anything/test?a=b -H 'apikey: auth-jack' -i

# 如果route绑定的service_id是2，因为登录consumer的白名单的service_id是1，不匹配则拒绝访问
HTTP/1.1 403 Forbidden
Date: Tue, 10 Jan 2023 07:27:59 GMT
Content-Type: text/plain; charset=utf-8
Transfer-Encoding: chunked
Connection: keep-alive
Server: APISIX/3.1.0

{"message":"The service_id is forbidden."}
```