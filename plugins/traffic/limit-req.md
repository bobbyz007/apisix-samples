作用： 限制单个客户端对服务的请求速率

1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - limit-req
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 在route 上配置

将客户端的 IP 地址作为限制请求速率的条件
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/anything/*",
    "plugins": {
        "limit-req": {
            "rate": 1,
            "burst": 2,
            "rejected_code": 503,
            "key_type": "var",
            "key": "remote_addr"
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

使用ab工具来测试
```shell
# 安装ab工具
sudo apt install apache2-utils

# 当不超过3次时，请求全部成功，有延时
ab -n 3 -c 3 http://127.0.0.1:9080/anything/test?a=b

# 当超过3次（rate+burst）时， 超过的请求会被拒绝（3个请求成功，有1个请求失败）
ab -n 4 -c 4 http://127.0.0.1:9080/anything/test?a=b
```

3. 在consumer上配置
limit-req 绑定在consumer上：
```shell
curl http://127.0.0.1:9180/apisix/admin/consumers \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "username": "consumer_jack",
    "plugins": {
        "key-auth": {
            "key": "auth-jack"
        },
        "limit-req": {
            "rate": 1,
            "burst": 2,
            "rejected_code": 403,
            "key": "consumer_name"
        }
    }
}'
```
指定路由配置key-auth:
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "methods": ["GET"],
    "uri": "/anything/*",
    "plugins": {
        "key-auth": {
            "key": "auth-jack"
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

使用ab工具来测试
```shell
# 当不超过3次时，请求全部成功，有延时
ab -n 3 -c 3 -H 'apikey: auth-jack' http://127.0.0.1:9080/anything/test?a=b 

# 当超过3次（rate+burst）时， 超过的请求会被拒绝（3个请求成功，有1个请求失败）
ab -n 4 -c 4 -H 'apikey: auth-jack' http://127.0.0.1:9080/anything/test?a=b
```
