#### 修改主机名

```shell
vi /etc/sysconfig/network
```

#### 修改IP地址

```shell
vi /etc/sysconfig/network-scripts/ifcfg-eth0
	DEVICE="eth0"
    TYPE="Ethernet"
    BOOTPROTO="static" # 启动方式
    HWADDR="00:0C:29:3C:BF:E7
    IPV6INIT="yes"
    NM_CONTROLLED="yes"
    ONBOOT="yes"
    UUID="ce22eeca-ecde-4536-8cc2-ef0dc36d4a8c"
    IPADDR="192.168.1.101" # IP地址
    NETMASK="255.255.255.0" # 子网掩码
    GATEWAY="192.168.1.1"  # 默认网关
```

> 克隆虚拟机：
>
> 1. 删除 HWADDR 和 UUID（重启后会自动生成）
>
> 2. `vi /etc/udev/rules.d/70-persistent-net.rules`删除 eth0，并将 eth1 改成 eth0。

#### 关闭防火墙

1. 查看防火墙状态：`service iptables status`
2. 关闭防火墙：`service iptables stop`
3. 查看防火墙开机启动状态：`chkconfig iptables --list`
4. 关闭防火墙开机启动：`chkconfig iptables off`

#### 添加hadoop用户及授予权限

```shell
useradd hduser # 添加用户
passwd hduser # 添加密码
su root
chmod u+w /etc/sudoers # 添加文件写权限
vi /etc/sudoers
	## Allow root to run any commands anywhere
	root    ALL=(ALL)       ALL
	hduser  ALL=(ALL)       ALL
chmod u-w /etc/sudoers # 撤销文件写权限
```

#### 修改hosts文件

```shell
vi /etc/hosts
	192.168.1.101 master01
	......
```

#### 配置SSH免登录

```shell
ssh-keygen -t rsa
ssh-copy-id master02（目标机器）
```

