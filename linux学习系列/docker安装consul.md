## 拉取Consul镜像

```shell
$ docker pull consul # 默认拉取latest
$ docker pull consul:1.6.1 # 拉取指定版本
```

## 安装并运行

```shell
docker run -d -p 8500:8500 --restart=always --name=consul consul:latest agent -server -bootstrap -ui -node=1 -client='0.0.0.0'
```

- agent: 表示启动 Agent 进程。

- server：表示启动 Consul Server 模式

- client：表示启动 Consul Cilent 模式。
- bootstrap：表示这个节点是 Server-Leader ，每个数据中心只能运行一台服务器。技术角度上讲 Leader 是通过 Raft 算法选举的，但是集群第一次启动时需要一个引导 Leader，在引导群集后，建议不要使用此标志。
- ui：表示启动 Web UI 管理器，默认开放端口 8500，所以上面使用 Docker 命令把 8500 端口对外开放。
- node：节点的名称，集群中必须是唯一的，默认是该节点的主机名。
- client：consul服务侦听地址，这个地址提供HTTP、DNS、RPC等服务，默认是127.0.0.1所以不对外提供服务，如果你要对外提供服务改成0.0.0.0
- join：表示加入到某一个集群中去。 如：-json=192.168.0.11。

## 关闭防火墙或开放8500端口

【查看防火墙是否开启】

```bash
$ systemctl status firewalld
```

【开启或关闭防火墙】

```bash
$ systemctl start firewalld
$ systemctl stop firewalld
```

【查看所有开启的端口】

```bash
$ firewall-cmd --list-ports
```

【开启80端口】

```bash
$ firewall-cmd --zone=public --add-port=8500/tcp --permanent
```

【重启防火墙，使其生效】

```bash
$ firewall-cmd --reload
```

如果是阿里云服务器，需要设置安全组：

来到实例管理页面，点击更多，点击网络和安全组，点击安全组配置。

![image-20201120164723821](img/docker%E5%AE%89%E8%A3%85consul/image-20201120164723821.png)

点击配置规则。

![image-20201120164850301](img/docker%E5%AE%89%E8%A3%85consul/image-20201120164850301.png)

点击添加安全组规则，端口范围改为8500。

![image-20201120165048181](img/docker%E5%AE%89%E8%A3%85consul/image-20201120165048181.png)

## 测试访问

访问：`hostname:8500/`

![image-20201120224802772](img/docker%E5%AE%89%E8%A3%85consul/image-20201120224802772.png)

1. services：放置服务
2. nodes：放置consul节点
3. key/value：放置一些配置信息
4. dc1：配置数据中心

## 参考

- https://cloud.tencent.com/developer/news/563375