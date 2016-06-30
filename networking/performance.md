# kubernetes网络性能
介绍kubernetes常见的网络模型和配置方法以及不同网络模型的性能评测

## 环境准备
* kubernetes master: 在一台虚拟机上部署kubernetes master相关组件。master本身不作为性能测试机，只作为集群的管理和协调者。
* kubernetes nodes: 在两台物理机上部署kubernetes worker节点的相关组件。
* 在Master和Node节点上，都安装部署CoreOS操作系统。
物理机配置：
24 core Intel(R) Xeon(R) CPU E5-2620 v2 @ 2.10GHz
64G memory
Intel Corporation I350 Gigabit Network Connection 千兆网卡（4口，评测中只使用第一个网口）

## 搭建kubernetes相关组件，使用flannel的host-gw网络模式
使用下面这个链接的cloud-config完成master和worker的配置：https://github.com/typhoonzero/kubernetes_binaries/tree/master/cloud-config
部署方式参考[这个](https://github.com/typhoonzero/kubernetes_binaries/blob/master/README.md)链接

## 性能评测
评测方法：
1. 跨机器ping延迟，物理机和在pod内的对比
1. 跨机器iperf3带宽测试
1. 跨机器nginx server使用apache benchmark(ab)测试

物理机之间的ping延迟：
```
ping 172.24.2.208
PING 172.24.2.208 (172.24.2.208) 56(84) bytes of data.
64 bytes from 172.24.2.208: icmp_seq=1 ttl=64 time=0.151 ms
64 bytes from 172.24.2.208: icmp_seq=2 ttl=64 time=0.121 ms
64 bytes from 172.24.2.208: icmp_seq=3 ttl=64 time=0.138 ms
64 bytes from 172.24.2.208: icmp_seq=4 ttl=64 time=0.147 ms
64 bytes from 172.24.2.208: icmp_seq=5 ttl=64 time=0.145 ms
64 bytes from 172.24.2.208: icmp_seq=6 ttl=64 time=0.116 ms
64 bytes from 172.24.2.208: icmp_seq=7 ttl=64 time=0.154 ms
```

在pod之间的ping延迟：
```
kubectl exec -it busybox -- ping 10.1.33.2
PING 10.1.33.2 (10.1.33.2): 56 data bytes
64 bytes from 10.1.33.2: seq=0 ttl=62 time=0.462 ms
64 bytes from 10.1.33.2: seq=1 ttl=62 time=0.267 ms
64 bytes from 10.1.33.2: seq=2 ttl=62 time=0.376 ms
64 bytes from 10.1.33.2: seq=3 ttl=62 time=0.269 ms
64 bytes from 10.1.33.2: seq=4 ttl=62 time=0.393 ms
64 bytes from 10.1.33.2: seq=5 ttl=62 time=0.312 ms
```

物理机之间的iperf3带宽：（使用toolbox, ```yum install -y iperf3```）
```
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  1.10 GBytes   942 Mbits/sec    0             sender
[  4]   0.00-10.00  sec  1.10 GBytes   941 Mbits/sec                  receiver

```

pod之间的iperf3带宽：
```
kubectl exec -it iperfbox2 -- iperf3 -c 10.1.66.5
...
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-10.00  sec  1.10 GBytes   944 Mbits/sec  134             sender
[  4]   0.00-10.00  sec  1.10 GBytes   941 Mbits/sec                  receiver
```

物理机之间的nginx ab结果：(使用docker --net=host启动nginx server， 在toolbox中使用ab)
```
Document Path:          /
Document Length:        612 bytes

Concurrency Level:      50
Time taken for tests:   6.808 seconds
Complete requests:      90000
Failed requests:        0
Total transferred:      76050000 bytes
HTML transferred:       55080000 bytes
Requests per second:    13220.57 [#/sec] (mean)
Time per request:       3.782 [ms] (mean)
Time per request:       0.076 [ms] (mean, across all concurrent requests)
Transfer rate:          10909.55 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.8      0       5
Processing:     1    3   1.7      3      14
Waiting:        0    3   1.7      3      14
Total:          1    4   1.7      3      15
WARNING: The median and mean for the initial connection time are not within a normal deviation
        These results are probably not that reliable.

Percentage of the requests served within a certain time (ms)
  50%      3
  66%      4
  75%      4
  80%      5
  90%      5
  95%      6
  98%      9
  99%     14
 100%     15 (longest request)
```

pod内的nginx性能：
(使用rc提交nginx pod并使用nodeport提交nginx service)
```
Document Path:          /
Document Length:        612 bytes

Concurrency Level:      50
Time taken for tests:   7.737 seconds
Complete requests:      90000
Failed requests:        0
Total transferred:      76050000 bytes
HTML transferred:       55080000 bytes
Requests per second:    11632.66 [#/sec] (mean)
Time per request:       4.298 [ms] (mean)
Time per request:       0.086 [ms] (mean, across all concurrent requests)
Transfer rate:          9599.22 [Kbytes/sec] received

Connection Times (ms)
              min  mean[+/-sd] median   max
Connect:        0    1   0.8      0       5
Processing:     1    4   1.9      3      14
Waiting:        1    3   2.0      3      14
Total:          2    4   1.8      4      15
WARNING: The median and mean for the initial connection time are not within a normal deviation
        These results are probably not that reliable.

Percentage of the requests served within a certain time (ms)
  50%      4
  66%      5
  75%      5
  80%      6
  90%      6
  95%      6
  98%      7
  99%     14
 100%     15 (longest request)
```

和之前使用flannel 的vxlan模式相比，性能大幅提升，真对nginx的评测结论，可以认为使用此模式可以作为生产环境使用。

# TODO
* 需要增加vxlan模式的评测数据
* 增加GCE方式的网络的配置方法
* 评测calico的网络性能
* L2模式的部署方案