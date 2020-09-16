# Docker 网络

## 环境准备

```shell
# 清空所有镜像
docker rmi -f $(docker images -qa)
# 查看当前网络
ip addr
见下图
# pull tomcat 运行容器
docker run -d -P --name tomcat01 tomcat
```

![image-20200904213016990](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200904213016990.png)



```shell
# 查看容器内部网络地址 , docker容器启动的时候默认分配的ip地址
docker exec -it tomcat01 ip addr	
```

![image-20200904213532662](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200904213532662.png)



```shell
# 测试Linux 宿主机是否能ping通docker 网络
ping 172.17.0.2

PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.064 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.043 ms
64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.186 ms
64 bytes from 172.17.0.2: icmp_seq=4 ttl=64 time=0.093 ms

结论：：Linux可以ping通docker容器内部
```

**原理**：每启动一个容器，docker就会给docker容器分配一个ip ，只要安装docker后，就会有一个网卡docker0，使用桥接模式和宿主机网卡通信，使用的技术使evth-pair技术

172.17.0.1  路由器地址  docker0 

172.17.0.2  Tomcat容器地址

```she
# 再次测试网络地址 ip addr
```

![image-20200904214743204](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200904214743204.png)



```shell
# 再次启动并pull tomcat容器 ，观察docker网络
docker run -d -P --name tomcat02 tomcat	
# 查看容器内部网络地址 , docker容器启动的时候默认分配的ip地址
docker exec -it tomcat02 ip addr	
```

![image-20200904215216569](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200904215216569.png)

新生成一对网卡

![image-20200904215258738](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200904215258738.png)

```she
# docker 之内的容器相互ping

[root@devops composetest]# docker exec -it tomcat02 ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.085 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.081 ms
64 bytes from 172.17.0.2: icmp_seq=3 ttl=64 time=0.089 ms

[root@devops composetest]# docker exec -it tomcat01 ping 172.17.0.3
PING 172.17.0.3 (172.17.0.3) 56(84) bytes of data.
64 bytes from 172.17.0.3: icmp_seq=1 ttl=64 time=0.041 ms
64 bytes from 172.17.0.3: icmp_seq=2 ttl=64 time=0.061 ms
64 bytes from 172.17.0.3: icmp_seq=3 ttl=64 time=0.054 ms
64 bytes from 172.17.0.3: icmp_seq=4 ttl=64 time=0.067 ms

# 结论，容器和容器之间是可以互相ping通的
```

网络通信原理图：

![image-20200904222305906](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200904222305906.png)

结论：tomcat01 和tomcat02 通过docker0网络进行路由，docker会默认给容器分配一个ip

![image-20200904222737851](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200904222737851.png)



## 容器之间互相通信

### --Link

``` shell
[root@devops composetest]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
075362640ab7        tomcat              "catalina.sh run"   11 hours ago        Up 11 hours         0.0.0.0:32769->8080/tcp   tomcat02
767331a35b83        tomcat              "catalina.sh run"   11 hours ago        Up 11 hours         0.0.0.0:32768->8080/tcp  
网络互相ping不通
[root@devops composetest]# docker exec -it tomcat01 ping tomcat02
ping: tomcat02: Name or service not known

[root@devops composetest]#  docker run -d -P --name tomcat03 --link tomcat02 tomcat
ef52c9ed92c303e1250985837f1a53c1f52680825e1a5ecf61530a6a84637e9c
[root@devops composetest]# docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS                     NAMES
ef52c9ed92c3        tomcat              "catalina.sh run"   5 seconds ago       Up 3 seconds        0.0.0.0:32770->8080/tcp   tomcat03
075362640ab7        tomcat              "catalina.sh run"   11 hours ago        Up 11 hours         0.0.0.0:32769->8080/tcp   tomcat02
767331a35b83        tomcat              "catalina.sh run"   11 hours ago        Up 11 hours         0.0.0.0:32768->8080/tcp   tomcat01

[root@devops composetest]# docker exec -it tomcat03 ping tomcat02
PING tomcat02 (172.17.0.3) 56(84) bytes of data.
64 bytes from tomcat02 (172.17.0.3): icmp_seq=1 ttl=64 time=0.062 ms
64 bytes from tomcat02 (172.17.0.3): icmp_seq=2 ttl=64 time=0.085 ms
64 bytes from tomcat02 (172.17.0.3): icmp_seq=3 ttl=64 time=0.133 ms

--Link 原理就是在tomcat03 的hosts 文件中添加了tomcat02的名字ip映射
[root@devops composetest]# docker exec -it tomcat03 cat /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.3      tomcat02 075362640ab7
172.17.0.4      72809b5628d5
```



### Docker network

Markdown uses email-style > characters for block quoting. They are presented as:

``` markdown
# 列出当前网络
[root@devops composetest]# docker network ls
NETWORK ID          NAME                  DRIVER              SCOPE
d891729fc784        bridge                bridge              local
c8a523f1c9ee        composetest_default   bridge              local
acaa25e7875b        host                  host                local
80b335a88a77        none                  null                local

# 插卡docker0 网络详细配置
[root@devops composetest]# docker network inspect d891729fc784
[
    {
        "Name": "bridge",
        "Id": "d891729fc7848ad48a984406a7e287335225cd3e94c12dcb26721d2c587b8149",
        "Created": "2020-08-30T20:16:17.620919029+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": null,
            "Config": [
                {
                    "Subnet": "172.17.0.0/16",
                    "Gateway": "172.17.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "075362640ab7c0e16f7b5dc1b6702c4d2872d98e231e315d7d3a61893662a7c8": {
                "Name": "tomcat02",
                "EndpointID": "79452d205124e1900d3e6df32705c044c9043a78485371b5f5ce108d3f6eef35",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            },
            "72809b5628d59267eacc2ecbbcf4ab34371a7696be9a1f94d8c95244ef647843": {
                "Name": "tomcat03",
                "EndpointID": "d027d033012211aa8a4de2ac0856b147e3d2ac9930936367f6b1e368f3ba7cbf",
                "MacAddress": "02:42:ac:11:00:04",
                "IPv4Address": "172.17.0.4/16",
                "IPv6Address": ""
            },
            "767331a35b83b87f8eca24b29365578bc9a0389a30861466af8e546ccb4919cc": {
                "Name": "tomcat01",
                "EndpointID": "2ceb539b5581ef3b89f86ff31fbfb0285dd803d5d536a44375b13839f508f06c",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
        "Options": {
            "com.docker.network.bridge.default_bridge": "true",
            "com.docker.network.bridge.enable_icc": "true",
            "com.docker.network.bridge.enable_ip_masquerade": "true",
            "com.docker.network.bridge.host_binding_ipv4": "0.0.0.0",
            "com.docker.network.bridge.name": "docker0",
            "com.docker.network.driver.mtu": "1500"
        },
        "Labels": {}
    }
]
```



### Docker 容器互联

docker0 默认不能使用域名访问，使用--link可以解决

``` shell
# 默认docker启动使用的docker0网络 ， 默认会添加参数  --net bridge
docker run -d -P tomcat01 --net bridge tomcat
```

### 自定义网络

生产实战中更多使用自定义网络满足需求:

``` shell
# 指定驱动默认是bridge ， 子网段，网关， 网络名
[root@devops _data]# docker network create --driver bridge --subnet 10.10.0.0/24 --gateway 10.10.0.1 mynet 
3a659d4a3da18b1b3045ba6792f2d61f7dbfe1bc474b3f983c7b019af08ad886

# 查看网络
[root@devops _data]# docker network ls
NETWORK ID          NAME                  DRIVER              SCOPE
d891729fc784        bridge                bridge              local
c8a523f1c9ee        composetest_default   bridge              local
acaa25e7875b        host                  host                local
3a659d4a3da1        mynet                 bridge              local
80b335a88a77        none                  null                local

# 查看网络详细配置
[root@devops _data]# docker network inspect mynet
[
    {
        "Name": "mynet",
        "Id": "3a659d4a3da18b1b3045ba6792f2d61f7dbfe1bc474b3f983c7b019af08ad886",
        "Created": "2020-09-05T10:34:17.035686377+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.10.0.0/24",
                    "Gateway": "10.10.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]

# 查看容器内部网络详情
[root@devops _data]# docker inspect  tomcat-net-02

# 查看mynet网络下，会出现tomcat01 和02 两个容器 
[root@devops _data]# docker network inspect mynet
[
    {
        "Name": "mynet",
        "Id": "3a659d4a3da18b1b3045ba6792f2d61f7dbfe1bc474b3f983c7b019af08ad886",
        "Created": "2020-09-05T10:34:17.035686377+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.10.0.0/24",
                    "Gateway": "10.10.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "9564f7b6c6c762c546e217b1ea6cc74b5b30391089cd2b4e5516c988d84bc8db": {
                "Name": "tomcat-net-02",
                "EndpointID": "9436c802f9527a53cb52d89b2236b1df470dbc2ee21c6b0b816fad06582b9407",
                "MacAddress": "02:42:0a:0a:00:03",
                "IPv4Address": "10.10.0.3/24",
                "IPv6Address": ""
            },
            "96f06148197c771fa751936f21d147e53be0aa7d63b40f1be22506bee7ad2151": {
                "Name": "tomcat-net-01",
                "EndpointID": "bccd9a6183887107dfb1f9c7ffcab4b03ca09d47f6f6a83f601d5031433e212e",
                "MacAddress": "02:42:0a:0a:00:02",
                "IPv4Address": "10.10.0.2/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]

# 自定义网络，容器之间可以相互ping
[root@devops _data]#  docker exec -it tomcat-net-02 ping 10.10.0.2
PING 10.10.0.2 (10.10.0.2) 56(84) bytes of data.
64 bytes from 10.10.0.2: icmp_seq=1 ttl=64 time=0.105 ms
64 bytes from 10.10.0.2: icmp_seq=2 ttl=64 time=0.110 ms

[root@devops _data]#  docker exec -it tomcat-net-01 ping 10.10.0.3 
PING 10.10.0.3 (10.10.0.3) 56(84) bytes of data.
64 bytes from 10.10.0.3: icmp_seq=1 ttl=64 time=0.046 ms
64 bytes from 10.10.0.3: icmp_seq=2 ttl=64 time=0.040 ms
```



### 网络连通

将属于不同网段的集群打通，通过容器连通其他网段默认网关

![image-20200905113736309](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200905113736309.png)



``` shell
# 常见tomcat 两台集群，默认使用docker0网络
[root@devops _data]# docker run -d -P --name tomcat01 tomcat 
48cfe3e914bdd72da6939550f1f73c5e1feee89d14c7e90efa72b762467a83a8
[root@devops _data]# docker run -d -P --name tomcat02 tomcat  
7db1b6817968752c28d99e9f1c4f4e95fa7a3034dc0d21f03c1b1c49cf5ab123

# 查看默认桥接网络下，已挂载 tomcat01 和 tomcat02 容器
[root@devops _data]# docker network inspect bridge
       "Containers": {
            "48cfe3e914bdd72da6939550f1f73c5e1feee89d14c7e90efa72b762467a83a8": {
                "Name": "tomcat01",
                "EndpointID": "485298a507978f2ec019b5221bfc394443220abceb565aabc49fb2e77b7bc8d3",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "7db1b6817968752c28d99e9f1c4f4e95fa7a3034dc0d21f03c1b1c49cf5ab123": {
                "Name": "tomcat02",
                "EndpointID": "332369f0766c897eede6c051cb7e4a7dabbdab6c4784d60d343c1a7b366c27c3",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
```

### 不同网段容器打通测试

``` shell
# 测试  tomcat01 ping tomcat-net-01
[root@devops _data]# docker exec -it tomcat01 ping tomcat-net-01
ping: tomcat-net-01: Name or service not known
[root@devops _data]# docker exec -it tomcat-net-01 ping tomcat01
ping: tomcat01: Name or service not known
测试失败
```

### Docker connect

![image-20200905114750698](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200905114750698.png)

```shell
# 打通命令
[root@devops _data]# docker network connect --help

Usage:  docker network connect [OPTIONS] NETWORK CONTAINER

Connect a container to a network

Options:
      --alias strings           Add network-scoped alias for the container
      --ip string               IPv4 address (e.g., 172.30.100.104)
      --ip6 string              IPv6 address (e.g., 2001:db8::33)
      --link list               Add link to another container
      --link-local-ip strings   Add a link-local address for the container

# 使用connect 将 mynet 网络连接上 tomcat01容器
[root@devops _data]# docker network connect mynet tomcat01

# 测试是否可以ping通
[root@devops _data]# docker exec -it tomcat01 ping tomcat-net-01
PING tomcat-net-01 (10.10.0.2) 56(84) bytes of data.
64 bytes from tomcat-net-01.mynet (10.10.0.2): icmp_seq=1 ttl=64 time=0.073 ms
64 bytes from tomcat-net-01.mynet (10.10.0.2): icmp_seq=2 ttl=64 time=0.070 ms

# 测试方向连接
[root@devops _data]# docker exec -it tomcat-net-01 ping tomcat01 
PING tomcat01 (10.10.0.4) 56(84) bytes of data.
64 bytes from tomcat01.mynet (10.10.0.4): icmp_seq=1 ttl=64 time=0.049 ms
64 bytes from tomcat01.mynet (10.10.0.4): icmp_seq=2 ttl=64 time=0.104 ms
成功， 网络通畅

# tomcat02 容器仍然ping不同，因为没有和mynet打通网络
[root@devops _data]# docker exec -it tomcat02 ping tomcat-net-01 
ping: tomcat-net-01: Name or service not known

# 查看自定义网络 mynet ， 发现tomcat01已经挂再该网络
[root@devops _data]# docker network inspect mynet
[
    {
        "Name": "mynet",
        "Id": "3a659d4a3da18b1b3045ba6792f2d61f7dbfe1bc474b3f983c7b019af08ad886",
        "Created": "2020-09-05T10:34:17.035686377+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.10.0.0/24",
                    "Gateway": "10.10.0.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "48cfe3e914bdd72da6939550f1f73c5e1feee89d14c7e90efa72b762467a83a8": {
                "Name": "tomcat01",
                "EndpointID": "336fa9278a91c70b8bb06a658943281213a4e1280aa1061e23dad7d29594619f",
                "MacAddress": "02:42:0a:0a:00:04",
                "IPv4Address": "10.10.0.4/24",
                "IPv6Address": ""
            },
            "9564f7b6c6c762c546e217b1ea6cc74b5b30391089cd2b4e5516c988d84bc8db": {
                "Name": "tomcat-net-02",
                "EndpointID": "9436c802f9527a53cb52d89b2236b1df470dbc2ee21c6b0b816fad06582b9407",
                "MacAddress": "02:42:0a:0a:00:03",
                "IPv4Address": "10.10.0.3/24",
                "IPv6Address": ""
            },
            "96f06148197c771fa751936f21d147e53be0aa7d63b40f1be22506bee7ad2151": {
                "Name": "tomcat-net-01",
                "EndpointID": "bccd9a6183887107dfb1f9c7ffcab4b03ca09d47f6f6a83f601d5031433e212e",
                "MacAddress": "02:42:0a:0a:00:02",
                "IPv4Address": "10.10.0.2/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
# mynet网段下，tomcat01 IP  10.10.0.4
[root@devops _data]# ping 10.10.0.4
PING 10.10.0.4 (10.10.0.4) 56(84) bytes of data.
64 bytes from 10.10.0.4: icmp_seq=1 ttl=64 time=0.114 ms
64 bytes from 10.10.0.4: icmp_seq=2 ttl=64 time=0.120 ms

# docker0 tomcat01 IP  172.17.0.2
[root@devops _data]# docker network inspect bridge
  "Containers": {
            "48cfe3e914bdd72da6939550f1f73c5e1feee89d14c7e90efa72b762467a83a8": {
                "Name": "tomcat01",
                "EndpointID": "485298a507978f2ec019b5221bfc394443220abceb565aabc49fb2e77b7bc8d3",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            },
            "7db1b6817968752c28d99e9f1c4f4e95fa7a3034dc0d21f03c1b1c49cf5ab123": {
                "Name": "tomcat02",
                "EndpointID": "332369f0766c897eede6c051cb7e4a7dabbdab6c4784d60d343c1a7b366c27c3",
                "MacAddress": "02:42:ac:11:00:03",
                "IPv4Address": "172.17.0.3/16",
                "IPv6Address": ""
            }
            
[root@devops _data]# ping 172.17.0.2
PING 172.17.0.2 (172.17.0.2) 56(84) bytes of data.
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.058 ms
64 bytes from 172.17.0.2: icmp_seq=2 ttl=64 time=0.064 ms

测试结果： tomcat01容器实现了跨网络，同一容器拥有两个IP
```

**结论**：docker中实现跨网络操作，就需要使用docker network connect 来实现。



### Redis 集群实战部署

**集群拓扑图**

三主三从六节点

![image-20200906215121875](C:\Users\wanghuan\AppData\Roaming\Typora\typora-user-images\image-20200906215121875.png)

### 搭建redis集群网络

``` shell
# 兴建redis集群网络 , 网段IP地址 10.10.10.0 - 10.10.10.255 ， 网关 10.10.10.1
[root@devops ~]# docker network create redis --subnet 10.10.10.0/24 --gateway 10.10.10.1
22cf5e325aca66282695c5e336809200e4b3ea99a610db4e38b188707cbdab98
# 查看redis 网卡是否创建成功
[root@devops ~]# docker network ls
NETWORK ID          NAME                  DRIVER              SCOPE
b353ce388de9        bridge                bridge              local
c8a523f1c9ee        composetest_default   bridge              local
acaa25e7875b        host                  host                local
3a659d4a3da1        mynet                 bridge              local
80b335a88a77        none                  null                local
22cf5e325aca        redis                 bridge              local

# 查看网络详情
[root@devops ~]# docker network inspect 22cf5e325aca
[
    {
        "Name": "redis",
        "Id": "22cf5e325aca66282695c5e336809200e4b3ea99a610db4e38b188707cbdab98",
        "Created": "2020-09-06T21:55:53.996960012+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.10.10.0/24",
                    "Gateway": "10.10.10.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {},
        "Options": {},
        "Labels": {}
    }
]
```



#### redis 配置文件

``` shell
# shell 脚本生成redis conf 文件
for port in $(seq 1 6)
  do 
    mkdir -p /opt/data/redis/node-${port}/conf
	touch  /opt/data/redis/node-${port}/conf/redis.conf
	cat << EOF > /opt/data/redis/node-${port}/conf/redis.conf
	port 6379 
	bind 0.0.0.0
	cluster-enabled yes
	cluster-config-file nodes.conf
	cluster-node-timeout 5000
	cluster-announce-ip 10.10.10.1${port}
	cluster-announce-port 6379
	cluster-announce-bus-port 16379
	appendonly yes
	EOF
  done
```



### 启动redis容器

```shell
redis1 - redis6 IP range 10.10.10.11 - 10.10.10.16
# 逐个启动 ， 首先启动redis1
[root@devops conf]#  docker run -p 6371:6379 -p 16371:16379 --name redis-1 \
> -v /opt/data/redis/node-1/data:/data \
> -v /opt/data/redis/node-1/conf/redis.conf:/etc/redis/redis.conf \
> -d --net redis --ip 10.10.10.11 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf
Unable to find image 'redis:5.0.9-alpine3.11' locally
5.0.9-alpine3.11: Pulling from library/redis
cbdbe7a5bc2a: Pull complete 
dc0373118a0d: Pull complete 
cfd369fe6256: Pull complete 
3e45770272d9: Pull complete 
558de8ea3153: Pull complete 
a2c652551612: Pull complete 
Digest: sha256:83a3af36d5e57f2901b4783c313720e5fa3ecf0424ba86ad9775e06a9a5e35d0
Status: Downloaded newer image for redis:5.0.9-alpine3.11
eb4177b2cf19f715c29a4f78cffcbbb1e6873860027b38a3f357bb6f4837f4b1

# 其余容器以此类推
docker run -p 6372:6379 -p 16372:16379 --name redis-2 \
-v /opt/data/redis/node-2/data:/data \
-v /opt/data/redis/node-2/conf/redis.conf:/etc/redis/redis.conf \
-d --net redis --ip 10.10.10.12 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf

docker run -p 6373:6379 -p 16373:16379 --name redis-3 \
-v /opt/data/redis/node-3/data:/data \
-v /opt/data/redis/node-3/conf/redis.conf:/etc/redis/redis.conf \
-d --net redis --ip 10.10.10.13 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf

docker run -p 6374:6379 -p 16374:16379 --name redis-4 \
-v /opt/data/redis/node-4/data:/data \
-v /opt/data/redis/node-4/conf/redis.conf:/etc/redis/redis.conf \
-d --net redis --ip 10.10.10.14 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf

docker run -p 6375:6379 -p 16375:16379 --name redis-5 \
-v /opt/data/redis/node-5/data:/data \
-v /opt/data/redis/node-5/conf/redis.conf:/etc/redis/redis.conf \
-d --net redis --ip 10.10.10.15 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf

docker run -p 6376:6379 -p 16376:16379 --name redis-6 \
-v /opt/data/redis/node-6/data:/data \
-v /opt/data/redis/node-6/conf/redis.conf:/etc/redis/redis.conf \
-d --net redis --ip 10.10.10.16 redis:5.0.9-alpine3.11 redis-server /etc/redis/redis.conf

# 查看容器启动
[root@devops conf]# docker ps
CONTAINER ID        IMAGE                    COMMAND                  CREATED             STATUS              PORTS                                              NAMES
a07f88e1edb6        redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   23 seconds ago      Up 21 seconds       0.0.0.0:6376->6379/tcp, 0.0.0.0:16376->16379/tcp   redis-6
38cd0498a4ea        redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   29 seconds ago      Up 27 seconds       0.0.0.0:6375->6379/tcp, 0.0.0.0:16375->16379/tcp   redis-5
d06b95063343        redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   33 seconds ago      Up 31 seconds       0.0.0.0:6373->6379/tcp, 0.0.0.0:16373->16379/tcp   redis-3
feed063791d4        redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   40 seconds ago      Up 38 seconds       0.0.0.0:6374->6379/tcp, 0.0.0.0:16374->16379/tcp   redis-4
050d07469da3        redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes        0.0.0.0:6372->6379/tcp, 0.0.0.0:16372->16379/tcp   redis-2
eb4177b2cf19        redis:5.0.9-alpine3.11   "docker-entrypoint.s…"   3 minutes ago       Up 3 minutes        0.0.0.0:6371->6379/tcp, 0.0.0.0:16371->16379/tcp   redis-1

# 查看redis 网络
[root@devops conf]# docker network inspect redis
[
    {
        "Name": "redis",
        "Id": "22cf5e325aca66282695c5e336809200e4b3ea99a610db4e38b188707cbdab98",
        "Created": "2020-09-06T21:55:53.996960012+08:00",
        "Scope": "local",
        "Driver": "bridge",
        "EnableIPv6": false,
        "IPAM": {
            "Driver": "default",
            "Options": {},
            "Config": [
                {
                    "Subnet": "10.10.10.0/24",
                    "Gateway": "10.10.10.1"
                }
            ]
        },
        "Internal": false,
        "Attachable": false,
        "Ingress": false,
        "ConfigFrom": {
            "Network": ""
        },
        "ConfigOnly": false,
        "Containers": {
            "050d07469da3f3df8cd7eee7698babcdf21a196cec757a24e885230c8d098289": {
                "Name": "redis-2",
                "EndpointID": "b5151fc06ee5b669d8f937daf059e873a20f681a47ee8ed4cb5f4f04d82c4db7",
                "MacAddress": "02:42:0a:0a:0a:0c",
                "IPv4Address": "10.10.10.12/24",
                "IPv6Address": ""
            },
            "38cd0498a4ea17096f7aafb0afaff9385fca4cbdfd1550c4fdcd2a6570d44e9c": {
                "Name": "redis-5",
                "EndpointID": "ba94efd2a9d9133726199574fa225cbcd59fc3cb0477cc950a0e357e0fc6726f",
                "MacAddress": "02:42:0a:0a:0a:0f",
                "IPv4Address": "10.10.10.15/24",
                "IPv6Address": ""
            },
            "a07f88e1edb6de447ec14a981bb69f5e7e21d945c74ca3df4663e7afb6e965c0": {
                "Name": "redis-6",
                "EndpointID": "939a5d927f8b405333cce09f4f5c83cb4a9ccc10e58d0c75cb04ff1dfe6499d0",
                "MacAddress": "02:42:0a:0a:0a:10",
                "IPv4Address": "10.10.10.16/24",
                "IPv6Address": ""
            },
            "d06b95063343149a8d99d18181732117b14f9d0201df44d9c365a0b08f13e8f8": {
                "Name": "redis-3",
                "EndpointID": "b4b6d9dfd534df474562e280b6eb1e94deb0fcc32e9b4acca132c212a60f70a7",
                "MacAddress": "02:42:0a:0a:0a:0d",
                "IPv4Address": "10.10.10.13/24",
                "IPv6Address": ""
            },
            "eb4177b2cf19f715c29a4f78cffcbbb1e6873860027b38a3f357bb6f4837f4b1": {
                "Name": "redis-1",
                "EndpointID": "d104f24a16776bba059681e1476df9930d95105c01544339fec806d101272296",
                "MacAddress": "02:42:0a:0a:0a:0b",
                "IPv4Address": "10.10.10.11/24",
                "IPv6Address": ""
            },
            "feed063791d4895eea7c08415e799fd0b4764317dcfd279be55e0af4d13a611c": {
                "Name": "redis-4",
                "EndpointID": "74c32a893ad898360847c6edab8cffaa374ec8527e361d69cd1f122d85b12158",
                "MacAddress": "02:42:0a:0a:0a:0e",
                "IPv4Address": "10.10.10.14/24",
                "IPv6Address": ""
            }
        },
        "Options": {},
        "Labels": {}
    }
]
```



### **创建redis 集群**

```shell
# 创建集群  进入redis 容器bash
[root@devops conf]# docker exec -it redis-1 /bin/sh

/data # redis-cli --cluster create 10.10.10.11:6379 10.10.10.12:6379 10.10.10.13
:6379 10.10.10.14:6379 10.10.10.15:6379 10.10.10.16:6379 --cluster-replicas 1
>>> Performing hash slots allocation on 6 nodes...
Master[0] -> Slots 0 - 5460
Master[1] -> Slots 5461 - 10922
Master[2] -> Slots 10923 - 16383
Adding replica 10.10.10.15:6379 to 10.10.10.11:6379
Adding replica 10.10.10.16:6379 to 10.10.10.12:6379
Adding replica 10.10.10.14:6379 to 10.10.10.13:6379
M: 3a2b0e9dd34b95345e32ed36ab993a1ad632b01d 10.10.10.11:6379
   slots:[0-5460] (5461 slots) master
M: 133a41ce53efc32ab3a5d52e7795afaa7471c13d 10.10.10.12:6379
   slots:[5461-10922] (5462 slots) master
M: 8da2f87e2615213ad72edbf2e522b763bae2332e 10.10.10.13:6379
   slots:[10923-16383] (5461 slots) master
S: 5c4551be394daaf82c90e9e2de9b8d26975c7cc7 10.10.10.14:6379
   replicates 8da2f87e2615213ad72edbf2e522b763bae2332e
S: 91c6f1045141df6848b8367b960331c2dea687ff 10.10.10.15:6379
   replicates 3a2b0e9dd34b95345e32ed36ab993a1ad632b01d
S: f715514fb3fbe1aa1880dbfd8bc49fe961d7f981 10.10.10.16:6379
   replicates 133a41ce53efc32ab3a5d52e7795afaa7471c13d
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join
...
>>> Performing Cluster Check (using node 10.10.10.11:6379)
M: 3a2b0e9dd34b95345e32ed36ab993a1ad632b01d 10.10.10.11:6379
   slots:[0-5460] (5461 slots) master
   1 additional replica(s)
M: 133a41ce53efc32ab3a5d52e7795afaa7471c13d 10.10.10.12:6379
   slots:[5461-10922] (5462 slots) master
   1 additional replica(s)
S: f715514fb3fbe1aa1880dbfd8bc49fe961d7f981 10.10.10.16:6379
   slots: (0 slots) slave
   replicates 133a41ce53efc32ab3a5d52e7795afaa7471c13d
S: 91c6f1045141df6848b8367b960331c2dea687ff 10.10.10.15:6379
   slots: (0 slots) slave
   replicates 3a2b0e9dd34b95345e32ed36ab993a1ad632b01d
M: 8da2f87e2615213ad72edbf2e522b763bae2332e 10.10.10.13:6379
   slots:[10923-16383] (5461 slots) master
   1 additional replica(s)
S: 5c4551be394daaf82c90e9e2de9b8d26975c7cc7 10.10.10.14:6379
   slots: (0 slots) slave
   replicates 8da2f87e2615213ad72edbf2e522b763bae2332e
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.

# 进去集群模式， 查看集群信息
/data # redis-cli -c
127.0.0.1:6379> cluster info
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_slots_pfail:0
cluster_slots_fail:0
cluster_known_nodes:6
cluster_size:3
cluster_current_epoch:6
cluster_my_epoch:1
cluster_stats_messages_ping_sent:288
cluster_stats_messages_pong_sent:286
cluster_stats_messages_sent:574
cluster_stats_messages_ping_received:281
cluster_stats_messages_pong_received:288
cluster_stats_messages_meet_received:5
cluster_stats_messages_received:574

# 查看主从机信息
127.0.0.1:6379>  cluster nodes
133a41ce53efc32ab3a5d52e7795afaa7471c13d 10.10.10.12:6379@16379 master - 0 1599404077299 2 connected 5461-10922
f715514fb3fbe1aa1880dbfd8bc49fe961d7f981 10.10.10.16:6379@16379 slave 133a41ce53efc32ab3a5d52e7795afaa7471c13d 0 1599404076292 6 connected
91c6f1045141df6848b8367b960331c2dea687ff 10.10.10.15:6379@16379 slave 3a2b0e9dd34b95345e32ed36ab993a1ad632b01d 0 1599404076292 5 connected
8da2f87e2615213ad72edbf2e522b763bae2332e 10.10.10.13:6379@16379 master - 0 1599404077502 3 connected 10923-16383
3a2b0e9dd34b95345e32ed36ab993a1ad632b01d 10.10.10.11:6379@16379 myself,master - 0 1599404075000 1 connected 0-5460
5c4551be394daaf82c90e9e2de9b8d26975c7cc7 10.10.10.14:6379@16379 slave 8da2f87e2615213ad72edbf2e522b763bae2332e 0 1599404076000 4 connected
```



### redis  HA测试

```Shell
# redis 内测试集群故障转移
127.0.0.1:6379> set a b
-> Redirected to slot [15495] located at 10.10.10.13:6379
OK
10.10.10.13:6379> get a
"b"
10.10.10.13:6379> get a
^C
/data # redis-cli -c
127.0.0.1:6379> get a
-> Redirected to slot [15495] located at 10.10.10.14:6379
"b"
10.10.10.14:6379> cluster nodes
8da2f87e2615213ad72edbf2e522b763bae2332e 10.10.10.13:6379@16379 master,fail - 1599404338366 1599404336553 3 connected
f715514fb3fbe1aa1880dbfd8bc49fe961d7f981 10.10.10.16:6379@16379 slave 133a41ce53efc32ab3a5d52e7795afaa7471c13d 0 1599404357503 6 connected
3a2b0e9dd34b95345e32ed36ab993a1ad632b01d 10.10.10.11:6379@16379 master - 0 1599404357000 1 connected 0-5460
91c6f1045141df6848b8367b960331c2dea687ff 10.10.10.15:6379@16379 slave 3a2b0e9dd34b95345e32ed36ab993a1ad632b01d 0 1599404358007 5 connected
5c4551be394daaf82c90e9e2de9b8d26975c7cc7 10.10.10.14:6379@16379 myself,master - 0 1599404356000 7 connected 10923-16383
133a41ce53efc32ab3a5d52e7795afaa7471c13d 10.10.10.12:6379@16379 master - 0 1599404357000 2 connected 5461-10922
```



### **Docker Compose**

**IDEA 创建SpringBoot微服务**

```shell
# 


```

