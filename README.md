# apisix-samples
Showing usage of apisix including kings of plugins, etc.

1. 启动apisix：docker方式
```shell
git clone https://github.com/apache/apisix-docker.git
cd apisix-docker/example

# 添加权限，否则apisix启动不了： https://github.com/apache/apisix-docker/issues/399
chmod -r 777 apisix_log/
docker compose -p docker-apisix up -d

# 启动或停止
docker compose -p docker-apisix start
docker compose -p docker-apisix stop
```

2. 创建本地测试http服务
```shell
docker run -p 1980:80 kennethreitz/httpbin
```

3. 查找apisix docker网桥ip，以访问外部宿主机的http服务，本例为 172.18.0.1:1980 
```shell
# 查询docker网桥列表
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ docker network ls
NETWORK ID     NAME                   DRIVER    SCOPE
2a4de33755e0   bridge                 bridge    local
248bf62cc170   docker-apisix_apisix   bridge    local
bc6e7f83650c   host                   host      local
e59deabb273f   none                   null      local
40698da19503   test-network           bridge    local

# 找到 NAME为docker-apisix_apisix 的网桥id： 248bf62cc170
# 网桥ip地址为 172.18.0.1
justin@KLVC-WXX9:~/workspace/apisix/apisix-docker/example$ ifconfig
br-248bf62cc170: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.1  netmask 255.255.0.0  broadcast 172.18.255.255
        inet6 fe80::42:14ff:feea:48d3  prefixlen 64  scopeid 0x20<link>
        ether 02:42:14:ea:48:d3  txqueuelen 0  (Ethernet)
        RX packets 2883  bytes 5532191 (5.5 MB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 3771  bytes 598975 (598.9 KB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
```