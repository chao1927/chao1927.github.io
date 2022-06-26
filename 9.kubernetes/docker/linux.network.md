# Linux Network

# 1. network namespace

## 1. 创建 network namespace

```shell
ip netns add netns1
```

## 2. 查看 network namespace

```
ip netns list
```

## 3. 删除 network namespace

```
ip netns delete netns1
```

## 4. 执行命令

```
ip netns exec netns1 ping 127.0.0.1

ip netns exec netns1 ip link list

ip netns exec netns1 ip link set dev lo up
```

## 5. 创建虚拟网卡

```
## 创建一对虚拟网卡
ip link add veth0 type veth peer name veth1

## 将其中一块虚拟网卡移动到 netns1 network namespace
ip link set veth1 netns netns1

## 设置虚拟网卡状态为 UP
ip netns exec netns1 ifconfig veth1 10.1.1.1/24 up
ifconfig veth0 10.1.1.2/24 up

## ping netns1/宿主机网络
ping 10.1.1.1
ip netns exec netns1 ping 10.1.1.2
ip netns exec netns1 ping www.baidu.com

## 测试路由表 防火墙
ip netns exec netns1 route
ip netns exec netns1 iptables -L
```

## 6. network namespace

```
非 root 进程被分配到 network namespace 后只能访问和配置已经存在于该 network namespace 的设备.
root 进程可以在 network namespace 里创建新的网络设备
network namespace 里的 root 进程还能把本 network namespace 的虚拟网络设备分配到其它 network namespace, 
这个操作路径可以从主机的根 network namespace 到用户自定义 network namespace, 反之亦然.

把 veth1 从 netns1 network namespace 移动到 root init 根 network namespace
ip netns exec netns1 ip link set veth1 netns 1

为了防范安全风险, 一般需要屏蔽这种行为
```

## 7. network namespace API 的使用

1. 创建 namespace 的黑科技: clone 系统调用

2. 维持 namespace 存在: /proc/PID/ns 目录的奥秘

```
/proc/PID/ns 目录下的文件还有一个作用, 当我们打开这些文件时, 只要文件描述符保存 open 状态, 对应的 namespace 就会一直存在
touch /my/net
mount --bind /proc/$$/ns/net /my/net
```

3. 往 namespace 里添加进程: setns 系统调用, 根据 namespace fd 进入指定 namespace

4. 帮助进程逃离 namespace: unshare 系统调用

 
# 2. veth pair

```
仅有 veth pair 设备, 容器是无法访问外部网络的. 从容器发出的数据包, 实际上是直接进入的 veth pair 设备的协议栈, 如果容器需要访问网络, 则需要使用网桥等技术
将 veth pair 设备接收的数据表通过某种方式转发出去
```

容器 network namespace 与 宿主机 的网络隧道

1. 方法1

```
## 容器
cat /sys/class/net/veth0/iflink

## 宿主机
cat /sys/class/net/vethxxxxx/iflink
```

2. 方法2

```
ip link show veth0
```

3. 方法3 

```
ethtool -S veth0
ip addr
```

# 3. Linux bridge

1. 创建 linux bridge

```
ip link add name br0 type bridge
ip link set br0 up
```

```
brctl addbr br0
```

2. 创建一对 veth 设备, 并配置 IP 地址

```
ip link add veth0 type veth peer name veth1
ip addr add 1.2.3.101/24 dev veth0
ip addr add 1.2.3.102/24 dev veth1
ip link set veth0 up
ip link set veth1 up
```

3. 将 veth0 连接到 br0 上

```
ip link set dev veth0 master br0
## 或者
brctl addif br0 veth0
```

4. 查看 bridge 上的网络设备

```
bridge link
brctl show
```

5. 调试

```
ping -c i -I veth0 1.2.3.102

tcpdump -n -i veth1
tcpdump -n -i veth0
tcpdump -n -i br0
```

6. 设置 bridge ip

```
ip addr del 1.2.3.101/24 dev veth0
ip addr add 1.2.3.101/24 dev br0

ping -c i -I br0 1.2.3.102
## ping 网关
ping -c i -I br0 1.2.3.1
```

