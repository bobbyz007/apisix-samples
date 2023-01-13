作用： 动态地将部分流量引导至各种上游服务。该插件可应用于灰度发布和蓝绿发布的场景。

1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - traffic-split
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 灰度发布

以下示例展示了如何通过配置 weighted_upstreams 的 weight 属性来实现流量分流。按 3:2 的权重流量比例进行划分，
其中 60% 的流量到达运行在 1991 端口上的上游服务，40% 的流量到达运行在 1990 端口上的上游服务：
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/anything/*",
    "plugins": {
        "traffic-split": {
            "rules": [
                {
                    "weighted_upstreams": [
                        {
                            "upstream": {
                                "name": "upstream_A",
                                "type": "roundrobin",
                                "nodes": {
                                    "172.18.0.1:1991":10
                                },
                                "timeout": {
                                    "connect": 15,
                                    "send": 15,
                                    "read": 15
                                }
                            },
                            "weight": 3
                        },
                        {
                            "weight": 2
                        }
                    ]
                }
            ]
        }
    },
    "upstream": {
            "type": "roundrobin",
            "nodes": {
                "172.18.0.1:1990": 1
            }
    }
}'
```

访问验证：
```shell
# 请求 5次，大概会有3次 命中1991端口的服务，2次命中1990端口的服务
ab -n 7 -c 7 http://127.0.0.1:9080/anything/test?a=b

```

3. 蓝绿发布
   在蓝绿发布场景中，你需要维护两个环境，一旦新的变化在蓝色环境（staging）中被测试和接受，
用户流量就会从绿色环境（production）转移到蓝色环境。
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/anything/*",
    "plugins": {
        "traffic-split": {
            "rules": [
                {
                    "match": [
                        {
                            "vars": [
                                ["http_release","==","new_release"]
                            ]
                        }
                    ],
                    "weighted_upstreams": [
                        {
                            "upstream": {
                                "name": "upstream_A",
                                "type": "roundrobin",
                                "nodes": {
                                    "172.18.0.1:1991":10
                                }
                            }
                        }
                    ]
                }
            ]
        }
    },
    "upstream": {
            "type": "roundrobin",
            "nodes": {
                "172.18.0.1:1990": 1
            }
    }
}'
```

访问验证：如果请求带有一个值为 new_release 的 release header，根据match规则它就会被引导至在插件上配置的新的上游服务：
```shell
# 引导到端口为1991的上游服务
curl http://127.0.0.1:9080/anything/test?a=b -H 'release: new_release' -i

# 引导到端口为1990的上游服务
curl http://127.0.0.1:9080/anything/test?a=b -H 'release: new_release' -i
```

4. 自定义发布一：单个vars规则
   配置了一个 vars 规则，流量按 3:2 的权重比例进行划分，不匹配 vars 的流量将被重定向到在路由上配置的上游服务。
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/anything/*",
    "plugins": {
        "traffic-split": {
            "rules": [
                {
                    "match": [
                        {
                            "vars": [
                                ["arg_name","==","jack"],
                                ["http_user-id",">","23"],
                                ["http_apisix-key","~~","[a-z]+"]
                            ]
                        }
                    ],
                    "weighted_upstreams": [
                        {
                            "upstream": {
                                "name": "upstream_A",
                                "type": "roundrobin",
                                "nodes": {
                                    "172.18.0.1:1991":10
                                }
                            },
                            "weight": 3
                        },
                        {
                            "weight": 2
                        }
                    ]
                }
            ]
        }
    },
    "upstream": {
            "type": "roundrobin",
            "nodes": {
                "172.18.0.1:1990": 1
            }
    }
}'
```
访问验证：如果match校验成功，将会有 60% 的请求被引导至插件 1991 端口的上游服务，
40% 的请求被引导至路由 1990 端口的上游服务：
```shell
# match 校验规则成功，按照权重3：2 大概分配到1991和1990端口的上游服务
ab -n 5 -c 5 -H 'user-id:30' -H 'apisix-key: hello' http://127.0.0.1:9080/anything/test?name=jack

# match 校验规则失败，全部分配到1990端口的上游服务
ab -n 5 -c 5 -H 'user-id:30' http://127.0.0.1:9080/anything/test?name=jack
```

5. 自定义发布二：多个vars规则，多个vars之间是或关系， vars内部是与关系。
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/anything/*",
    "plugins": {
        "traffic-split": {
            "rules": [
                {
                    "match": [
                        {
                            "vars": [
                                ["arg_name","==","jack"],
                                ["http_user-id",">","23"],
                                ["http_apisix-key","~~","[a-z]+"]
                            ]
                        },
                        {
                            "vars": [
                                ["arg_name2","==","rose"],
                                ["http_user-id2","!",">","33"],
                                ["http_apisix-key2","~~","[a-z]+"]
                            ]
                        }
                    ],
                    "weighted_upstreams": [
                        {
                            "upstream": {
                                "name": "upstream_A",
                                "type": "roundrobin",
                                "nodes": {
                                    "172.18.0.1:1991":10
                                }
                            },
                            "weight": 3
                        },
                        {
                            "weight": 2
                        }
                    ]
                }
            ]
        }
    },
    "upstream": {
            "type": "roundrobin",
            "nodes": {
                "172.18.0.1:1990": 1
            }
    }
}'
```
访问验证： match的多个vars之间之间是 或 关系， vars内部表达式之间是 与 关系。
```shell
# 两个var都校验成功，按照权重3：2 大概分配到1991和1990端口的上游服务
ab -n 5 -c 5 -H 'user-id:30' -H 'user-id2:22' -H 'apisix-key: hello' -H 'apisix-key2: world' http://127.0.0.1:9080/anything/test?name=jack&name2=rose
# 第一个var校验成功，按照权重3：2 大概分配到1991和1990端口的上游服务
ab -n 5 -c 5 -H 'user-id:30' -H 'user-id2:100' -H 'apisix-key: hello' -H 'apisix-key2: world' http://127.0.0.1:9080/anything/test?name=jack
# 第二个var校验成功，按照权重3：2 大概分配到1991和1990端口的上游服务
ab -n 5 -c 5 -H 'user-id:5' -H 'user-id2:22' -H 'apisix-key: hello' -H 'apisix-key2: world' http://127.0.0.1:9080/anything/test?name2=rose

# 两个var都校验失败，全部分配到1990端口的上游服务
ab -n 5 -c 5 http://127.0.0.1:9080/anything/test?name=jack
```

6. 自定义发布三：配置多个rules
   配置多个 rules 属性，实现不同的匹配规则与上游一一对应。当请求头 x-api-id 为 1 时，请求会被引导至 1991 端口的上游服务；
当 x-api-id 为 2 时，请求会被引导至 1992 端口的上游服务；否则请求会被引导至 1990 端口的上游服务：
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1 \
-H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/anything/*",
    "plugins": {
        "traffic-split": {
            "rules": [
                {
                    "match": [
                        {
                            "vars": [
                                ["http_x-api-id","==","1"]
                            ]
                        }
                    ],
                    "weighted_upstreams": [
                        {
                            "upstream": {
                                "name": "upstream-A",
                                "type": "roundrobin",
                                "nodes": {
                                    "172.18.0.1:1991":1
                                }
                            },
                            "weight": 3
                        }
                    ]
                },
                {
                    "match": [
                        {
                            "vars": [
                                ["http_x-api-id","==","2"]
                            ]
                        }
                    ],
                    "weighted_upstreams": [
                        {
                            "upstream": {
                                "name": "upstream-B",
                                "type": "roundrobin",
                                "nodes": {
                                    "172.18.0.1:1992":1
                                }
                            },
                            "weight": 3
                        }
                    ]
                }
            ]
        }
    },
    "upstream": {
            "type": "roundrobin",
            "nodes": {
                "172.18.0.1:1990": 1
            }
    }
}'
```
验证访问：
```shell
# 请求头为1的时候，命中1991端口的上游服务
curl http://127.0.0.1:9080/anything/test?a=b -H 'x-api-id: 1'

# 请求头为2的时候，命中1992端口的上游服务
curl http://127.0.0.1:9080/anything/test?a=b -H 'x-api-id: 2'

# 请求头不为1或2的时候，命中1990端口的上游服务
curl http://127.0.0.1:9080/anything/test?a=b -H 'x-api-id: 3'
```
