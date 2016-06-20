# Flannel Tutorial

## 简介
Flannel是一个使用overlay方式的虚拟网络的实现。可以为每个host分配一个网段，容器(Pod)运行时即可自动从这个网段分配IP地址。Flannel是解决Kubernetes网络的一个常用的方式。

<img src="./flannel-arch.png" width=800 />
如上图所示，每台服务器上会启动flanneld的一个守护进程，并为本机分配一个单独的子网(图中的10.1.16.1/24)。flanneld进程启动的时候，
会从etcd中获取预先配置好的overlay network的网络配置(10.1.0.0/16)，并获取目前已经使用过的子网，然后为本机生成一个专属的子网。
子网生成完毕之后，生成subnet.env文件，包含对应的子网信息。coreos使用flanneld生成的subnet.env来配置/run/flannel_docker_opts.env
是docker daemon运行在这个子网，即docker生成的网桥docker0配置在本子网(--bip=10.1.16.1/24)。这样kubernetes 管理Pod的启动就会自动在
这个网段下分配IP。

## 在coreos中运行flannel

flanneld 依赖etcd作为配置中心，参考[这篇](https://coreos.com/etcd/docs/latest/clustering.html)文章来先完成etcd集群的部署和搭建。

完成etcd安装和启动之后，先确保etcd服务可以正确的通过http和etcdctl访问

使用下面的命令，为flanneld配置网络和模式。
* 替换```$POD_NETWORK```为实际需要配置的网段CIDR
* 替换```$ETCD_SERVER```为etcd的地址
```
curl -X PUT -d "value={\"Network\":\"$POD_NETWORK\",\"Backend\":{\"Type\":\"vxlan\"}}" "$ETCD_SERVER/v2/keys/coreos.com/network/config"
```
创建/etc/flannel/options.env文件，写入下面的配置信息：
* 替换```${ADVERTISE_IP}```为本机对外可访问的ip地址
* 替换```${ETCD_ENDPOINTS}```为etcd集群的访问地址
```
FLANNELD_IFACE=${ADVERTISE_IP}
FLANNELD_ETCD_ENDPOINTS=${ETCD_ENDPOINTS}
```
创建systemd的drop-in文件/etc/systemd/system/flanneld.service.d/40-ExecStartPre-symlink.conf，在flannel service启动的时候拷贝配置文件到对应位置：
```
[Service]
ExecStartPre=/usr/bin/ln -sf /etc/flannel/options.env /run/flannel/options.env
```
配置docker service依赖flannel启动，创建文件/etc/systemd/system/docker.service.d/40-flannel.conf
```
[Unit]
Requires=flanneld.service
After=flanneld.service
```
完成以上的配置之后，就可以使用systemctl start flanneld启动服务。如果docker此时也是未启动状态，可以直接systemctl start docker就可以
把flanneld和docker daemon同时启动。

## 配置使用不同的网络模式
flannel 有多种网络模式可以配置，
* udp模式
使用udp协议完成overlay封包。不推荐使用此模式，性能较差
* vxlan模式
直接使用linux kernel的vxlan功能封包。经过测试此模式性能损失也很大
* host-gw模式
原理和GCE模式类似，直接将发忘Pod子网的数据路由到对应的host。要求host之间有直接的L2连接。性能不错
* aws-vpc, gce模式
对应只在aws和gce环境下使用
* alloc只分配网络，不做包转发

flannel的网络模式，在初始化集群配置etcd的时候就需要设置好，参考上面的步骤。
