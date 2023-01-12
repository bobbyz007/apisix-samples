作用： 提供了镜像客户端请求的能力

1. 激活插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - proxy-mirror
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 启动2个服务，分别监听端口1980，1990，模拟镜像服务
```shell
# start httpbin service, listen on http 1980
docker run --rm -p 1980:80 kennethreitz/httpbin

# start http echo service: listen on http 1990
# see https://github.com/mendhak/docker-http-https-echo
docker run -p 1990:8080 -p 18443:8443 --rm -t mendhak/http-https-echo:28
```

3. 在route 上配置

将客户端的 IP 地址作为限制请求速率的条件
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1  \
  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "plugins": {
        "proxy-mirror": {
           "host": "http://172.18.0.1:1990"
        }
    },
    "upstream": {
        "nodes": {
            "172.18.0.1:1980": 1
        },
        "type": "roundrobin"
    },
    "uri": "/anything/*"
}'
```

3. 访问验证
```shell
curl -i http://127.0.0.1:9080/anything/test?a=b

# 可以在http-echo 镜像服务上查看到访问记录，说明镜像服务访问成功：
{
    "path": "/anything/test",
    "headers": {
        "host": "127.0.0.1:9080",
        "connection": "close",
        "user-agent": "curl/7.68.0",
        "accept": "*/*"
    },
    "method": "GET",
    "body": "",
    "fresh": false,
    "hostname": "127.0.0.1",
    "ip": "::ffff:172.17.0.1",
    "ips": [],
    "protocol": "http",
    "query": {
        "a": "b"
    },
    "subdomains": [],
    "xhr": false,
    "os": {
        "hostname": "e4eab16979dd"
    },
    "connection": {}
}
::ffff:172.17.0.1 - - [12/Jan/2023:07:51:39 +0000] "GET /anything/test?a=b HTTP/1.1" 200 435 "-" "curl/7.68.0"
```
