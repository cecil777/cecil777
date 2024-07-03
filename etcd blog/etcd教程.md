etcd 是一个开源的、分布式的、强一致性的、可靠的键值存储系统。
常用于存储分布式系统的关键数据。
它可以在网络分区期间可以优雅地处理leader选举，并且可以容忍机器故障。

**关键特性**

- 分布式
- 强一致性
- 键值存储
- 监听数据变化

### 键值存储

etcd的存储格式，仅支持键值（key-value）存储，etcd的键（key）以目录树结构方式组织，就是key的命名和存储类似我们的目录结构。

key的命名例子：

1. /tizi365
2. /tizi365/site/name
3. /tizi365/site/domain
4. /tizi365/status/urls

key以这种目录树结构方式存储，etcd支持前缀搜索，例如：搜索key以/tizi365为前缀的所有键值。

### 强一致性

etcd通过Raft协议保证etcd各个节点数据的一致性，任何时刻，都可以从任意etcd节点查询到正确数据。

### 监听数据变化

支持监听某个key，或者某一批key的数据变化，当这些key的数据发生变化，就会立即通知监听客户端。

### etcd和redis的差异

etcd和redis都支持键值存储，也支持分布式特性，redis支持的数据格式更加丰富。但是他们两个的定位和应用场景不一样，关键差异如下：

- redis在分布式环境下不是强一致性的，可能会丢失数据，或者读不到最新数据。
- redis的数据变化监听机制没有etcd完善。
- 因为etcd的强一致性机制，导致etcd的性能不如redis。

基于上面的关键差异，如果系统没有强一致性的要求，需要缓存系统，redis比较合适。
如果需要存储分布式系统的元数据，辅助分布式系统协调通知、关键配置，etcd比较合适。
这些场景对读写的吞吐量没有缓存要求那么高，但是对数据一致性要求比较高。
例如我们开发一个分布式系统。是主从架构，需要实现自动选举一个节点作为主节点。
这个选举状态数据的存储引擎，必须是高可用、强一致性的，否则每个节点读取到的状态和数据都不一致、或者读取不到数据，集群就乱了，不知道谁是主节点。

etcd和Zookeeper是定位类似的项目，和redis定位不一样。

### 应用场景

- 分布式系统配置管理
- 服务注册与发现
- 选主，就是选举leader
- 应用调度
- 分布式锁

## etcd单机部署

作为本地开发和测试环境，我们不需要部署etcd集群，只要部署一个etcd实例即可。

### 下载安装包

到etcd的github地址，下载最新的安装包。

``` go
https://github.com/etcd-io/etcd/releases/
```

安装版本举例说明

- etcd-版本号-darwin-amd64.zip - macos版本
- etcd-版本号-linux-amd64.tar.gz - linux 64位版本
- etcd-版本号-windows-amd64.zip - windows 64位版本

解压缩包后，将得到类似的目录结构:

etcd-v3.2.28-darwin-amd64/

├── Documentation - etcd文档目录

├── etcd - etcd服务端程序

└── etcdctl - etcd客户端程序，用来操作服务端

### 启动etcd

切换到etcd安装目录，下面以Linux为例子

``` bash
./etcd
```

打开命令窗口直接运行etcd程序，就可以启动默认配置的etcd服务器。

启动etcd输出类似:

``` bash
jogindembp:etcd-v3.2.28-darwin-amd64 jogin$ ./etcd
2019-11-14 23:11:46.531199 I | etcdmain: etcd Version: 3.2.28
2019-11-14 23:11:46.531305 I | etcdmain: Git SHA: 2d861f39e
2019-11-14 23:11:46.531312 I | etcdmain: Go Version: go1.8.7
2019-11-14 23:11:46.531318 I | etcdmain: Go OS/Arch: darwin/amd64
........忽略.....
2019-11-14 23:11:46.533058 I | embed: listening for client requests on localhost:2379
```

提示：etcd服务端处理请求的默认端口是2379

### 测试etcd

我们可以通过安装目录的etcdctl命令测试，etcd是否启动成功。

例子:

切换到安装目录, 执行下面命令

``` bash
./etcdctl set /config/title tizi365
```

如果正常的话，会输出：

``` bash
tizi365
```

提示：为了方便调试，可以将etcd的安装目录添加到PATH环境变量中，就不需要每次都要切换到etcd安装目录，执行命令。

### 关闭etcd服务

只要杀掉etcd进程既可。

例如：

``` bash
# 假如60999是etcd进程id
kill 60999
```

注意：不要使用kill -9 杀掉进程，可能会导致etcd丢失数据。

## etcd 集群部署

etcd集群部署，通常至少部署3个etcd节点(推荐奇数个节点)，下面一步步接受集群搭建方法。

提示：如何安装etcd，请参考etcd单机部署章节。

### 集群规划

这里规划一个由3台服务器节点组成的etcd集群，如下表：

| 节点名字   | 服务器ip     |
|--------|-----------|
| infra0 | 10.0.1.10 |
| infra1 | 10.0.1.11 |
| infra2 | 10.0.1.12 |

### etcd关键参数说明

| 参数                            | 说明                                                                                         |
|-------------------------------|--------------------------------------------------------------------------------------------|
| --name                        | etcd节点名字                                                                                   |
| --initial-cluster             | etcd启动的时候，通过这个配置找到其他ectd节点的地址列表，格式：'节点名字1=<http://节点ip1:2380,节点名字1=http://节点ip1:2380>,.....' |
| --initial-cluster-state       | 初始化的时候，集群的状态 "new" 或者 "existing"两种状态，new代表新建的集群，existing表示加入已经存在的集群。                       |
| --listen-client-urls          | 监听客户端请求的地址列表，格式：'<http://localhost:2379>', 多个用逗号分隔。                                          |
| --advertise-client-urls       | 如果--listen-client-urls配置了，多个监听客户端请求的地址，这个参数可以给出，建议客户端使用什么地址访问etcd。                         |
| --listen-peer-urls            | 服务端节点之间通讯的监听地址，格式：'<http://localhost:2380>'                                                  |
| --initial-advertise-peer-urls | 建议服务端之间通讯使用的地址列表。                                                                          |

### 启动节点1

``` bash
$ etcd --name infra0 --initial-advertise-peer-urls http://10.0.1.10:2380 \
  --listen-peer-urls http://10.0.1.10:2380 \
  --listen-client-urls http://10.0.1.10:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.10:2379 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
```

### 启动节点2

``` bash
$ etcd --name infra1 --initial-advertise-peer-urls http://10.0.1.11:2380 \
  --listen-peer-urls http://10.0.1.11:2380 \
  --listen-client-urls http://10.0.1.11:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.11:2379 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
```

### 启动节点3

``` bash
$ etcd --name infra2 --initial-advertise-peer-urls http://10.0.1.12:2380 \
  --listen-peer-urls http://10.0.1.12:2380 \
  --listen-client-urls http://10.0.1.12:2379,http://127.0.0.1:2379 \
  --advertise-client-urls http://10.0.1.12:2379 \
  --initial-cluster infra0=http://10.0.1.10:2380,infra1=http://10.0.1.11:2380,infra2=http://10.0.1.12:2380 \
  --initial-cluster-state new
```

提示：每台服务器启动的etcd实例，--initial-cluster 参数都是一样的，列出整个集群所有节点的服务端通讯地址。

### 开机启动

上面是直接在命令行启动etcd实例，关闭命令窗口，etcd就退出了，推荐使用进程管理软件，启动etcd，例如：centos系统，使用systemd启动etcd，具体如何配置网上找一下systemd的资料即可。

## etcd 命令行操作

本章主要介绍通过etcdctl命令操作etcd服务，读写数据、监听数据。

etcdctl是etcd安装包自带的一个客户端工具，可以通过etcdctl操作etcd服务。

### etcdctl关键参数

输入下面命令可以查看etcdctl命令的帮助信息。

``` bash
etcdctl -h
```

etcdctl最关键的参数就是etcd服务的地址是什么？

可以通过 --endpoints参数指定etcd的客户端监听地址列表。

例如：

``` bash
etcdctl  --endpoints "http://127.0.0.1:2379,http://127.0.0.1:4001" 子命令
```

如果你的etcd安装在本地，可以不需要手动指定--endpoints参数。

### 设置key

命令格式：

``` bash
etcdctl set key value
```

例子:

``` bash
etcdctl set /config/name tizi365
```

### 读取key

命令格式：

``` bash
etcdctl get key
```

例子:

``` bash
# 查询一个key
$ etcdctl get /config/name
#输出
tizi365
```

### 删除key

命令格式:

``` bash
etcdctl rm key
```

例子：

``` bash
etcdctl rm foo
```

### 监听key

etcd支持监听key的数据变化

命令格式：

``` bash
etcdctl watch [-f] [-r] key
```

可选参数说明：

- -f 除非输入CTRL+C否则一直监控，不退出

- -r 监听key包括key的所有子目录下的数据

例子：

``` bash
# 监听/config这个目录下的所有内容
$ etcdctl watch -f -r /config

# 输出例子，可以看到/config目录下所有的key的写入操作都被监控到
[set] /config/name
1232
[set] /config/ttl
1232


# 监控指定的key
$ etcdctl watch -f /config/name
```

## Go连接etcd

本章介绍go如何连接etcd服务。

### 1. 安装依赖

``` go
go get go.etcd.io/etcd/clientv3
```

### 2.go操作etcd步骤

1. 通过clientv3.New创建etcd客户端，连接etcd。
2. 通过步骤1创建的etcd客户端操作etcd。
3. 关闭etcd连接。

### 3.连接etcd

通过clientv3.New创建一个etcd客户端

``` go
// 通过clientv3.Config配置，客户端参数
cli, err := clientv3.New(clientv3.Config{
// etcd服务端地址数组，可以配置一个或者多个
Endpoints:   []string{"localhost:2379", "localhost:22379", "localhost:32379"},
// 连接超时时间，5秒
DialTimeout: 5 * time.Second,
})

if err != nil {
// 错误处理
}

// ...对etcd进行crud操作....

// 延迟关闭客户端，记得用完后关闭客户端
defer cli.Close()
```

## Go etcd增删改查（CRUD操作）

etcd是一个键值存储系统，类似ZooKeeper, key是以目录结构形式组织的，如下：

key的命名例子:

``` bash
/tizi365
/tizi365/site/name
/tizi365/site/domain
/tizi365/status/urls
```

key以这种目录树结构方式存储，etcd支持前缀搜索，例如：搜索key 以 /tizi365 为前缀的所有键值。

下面介绍golang对etcd的基本操作。

### 1.写入数据

通过Put函数写入数据，如果Key存在则覆盖，否则新建一个。

``` go
cli, err := clientv3.New(...省略...)
if err != nil {
log.Fatal(err)
}
defer cli.Close()

// 获取上下文，设置请求超时时间为5秒
ctx, _ := context.WithTimeout(context.Background(), 5 * time.Second)
// 设置key="/tizi365/url" 的值为 www.tizi365.com
_, err = cli.Put(ctx, "/tizi365/url", "www.tizi365.com")

if err != nil {
log.Fatal(err)
}
```

提示，具体如何连接etcd请参考, 连接etcd章节。

### 2.查询数据

通过Get函数，可以查询key的值

``` go
cli, err := clientv3.New(...省略...)
if err != nil {
log.Fatal(err)
}
defer cli.Close()

// 获取上下文，设置请求超时时间为5秒
ctx, _ := context.WithTimeout(context.Background(), 5 * time.Second)
// 读取key="/tizi365/url" 的值
resp, err := cli.Get(ctx, "/tizi365/url")

if err != nil {
log.Fatal(err)
}

// 虽然这个例子我们只是查询一个Key的值，
// 但是Get的查询结果可以表示多个Key的结果例如我们根据Key进行前缀匹配,Get函数可能会返回多个值。
for _, ev := range resp.Kvs {
fmt.Printf("%s : %s\n", ev.Key, ev.Value)
}
```

### 3.前缀匹配

etcd支持key前缀匹配，Get,Delele函数都支持前缀匹配，只需要添加clientv3.WithPrefix()参数即可。

例子:

``` go
cli, err := clientv3.New(...省略...)
if err != nil {
log.Fatal(err)
}
defer cli.Close()

// 获取上下文，设置请求超时时间为5秒
ctx, _ := context.WithTimeout(context.Background(), 5 * time.Second)

// 读取key前缀等于"/tizi365/"的所有值
resp, err := cli.Get(ctx, "/tizi365/", clientv3.WithPrefix())

if err != nil {
log.Fatal(err)
}

// 遍历查询结果
for _, ev := range resp.Kvs {
fmt.Printf("%s : %s\n", ev.Key, ev.Value)
}
```

### 4.删除数据

通过Delete函数删除数据

``` go
cli, err := clientv3.New(...省略...)
if err != nil {
    log.Fatal(err)
}
defer cli.Close()

// 获取上下文，设置请求超时时间为5秒
ctx, _ := context.WithTimeout(context.Background(), 5 * time.Second)
// 删除key="/tizi365/url" 的值
_, err = cli. Delete(ctx, "/tizi365/url")

if err != nil {
    log.Fatal(err)
}

// 批量删除key以"/tizi365/"为前缀的值
// 加上clientv3.WithPrefix()参数代表key前缀匹配的意思
_, err = cli. Delete(ctx, "/tizi365/", clientv3.WithPrefix())
if err != nil {
    log.Fatal(err)
}
```

## Go etcd watch监控数据

etcd的核心特性之一，就是我们可以监控key的数据变化，只要有人修改了key的值，我们都可以监控到变化的值。

### 监控指定Key

``` go
cli, err := clientv3.New(...忽略...)
if err != nil {
    log.Fatal(err)
}
defer cli.Close()

// 监控key=/tizi 的值
rch := cli.Watch(context.Background(), "/tizi")
// 通过channel遍历key的值的变化
for wresp := range rch {
    for _, ev := range wresp.Events {
        fmt.Printf("%s %q : %q\n", ev.Type, ev.Kv.Key, ev.Kv.Value)
    }
}
```

提示，具体如何连接etcd请参考, 连接etcd章节。

### 根据key前缀监控一组key的值

``` go
cli, err := clientv3.New(...忽略...)
if err != nil {
    log.Fatal(err)
}
defer cli.Close()

// 监控以/tizi为前缀的所有key的值
rch := cli.Watch(context.Background(), "/tizi", clientv3.WithPrefix())
// 通过channel遍历key的值的变化
for wresp := range rch {
    for _, ev := range wresp.Events {
        fmt.Printf("%s %q : %q\n", ev.Type, ev.Kv.Key, ev.Kv.Value)
    }
}
```
