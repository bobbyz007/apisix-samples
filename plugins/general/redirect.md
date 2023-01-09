1. 激活redirect插件并重启apisix
```shell
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ vi apisix_conf/config.yaml 

# 在文件末尾添加配置
plugins:
  - redirect
```
重启apisix：
```shell
docker compose -p docker-apisix restart
```

2. 创建route
```shell
curl http://127.0.0.1:9180/apisix/admin/routes/1  -H 'X-API-KEY: edd1c9f034335f136f87ad84b625c8f1' -X PUT -d '
{
    "uri": "/test/index.html",
    "plugins": {
        "redirect": {
            "uri": "/anything/test?a=b",
            "ret_code": 301
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
curl http://127.0.0.1:9080/test/index.html -i
```
返回结果显示跳转：Location指示目标uri
```shell
HTTP/1.1 301 Moved Permanently
Date: Mon, 09 Jan 2023 07:27:50 GMT
Content-Type: text/html
Content-Length: 241
Connection: keep-alive
Location: /anything/test?a=b
Server: APISIX/3.1.0

<html>
<head><title>301 Moved Permanently</title></head>
<body>
<center><h1>301 Moved Permanently</h1></center>
<hr><center>openresty</center>
<p><em>Powered by <a href="https://apisix.apache.org/">APISIX</a>.</em></p></body>
</html>
```