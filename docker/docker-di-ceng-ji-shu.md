# Docker 底层技术

## chroot

在类 Unix 操作系统中，根目录\(root directory\) `/` 是顶级目录，所有文件系统的路径都是从跟 `/` 开始，使用 Chroot 可以更改进程及子进程识别到的根目录 `/` 改为其他目录。

这里我们通过 [sig-cloud-instance-build](https://github.com/CentOS/sig-cloud-instance-build.git) \(centos 的构建仓库\) 来生成一个最小化的 rootfs ，在以下示例中会用到，如果服务器不支持虚拟化，[sig-cloud-instance-images](https://github.com/CentOS/sig-cloud-instance-images.git) 中有构建好的镜像进行下载。

```text
yum -y install libvirt-python lorax libvirt virt-install 
systemctl start libvirtd
pip install setuptools --upgrade
pip install requests urllib3 pyOpenSSL --force --upgrade
mkdir test && cd test
curl -LO https://mirrors.aliyun.com/centos/7/os/x86_64/images/boot.iso
curl -LO https://raw.githubusercontent.com/CentOS/sig-cloud-instance-build/master/docker/centos-7-x86_64.ks
livemedia-creator --make-tar --iso=./boot.iso --ks=./centos-7-x86_64.ks --image-name=centos-7-docker.tar.xz
```

将构建出的 centos-7-x86\_64-docker.tar.xz 解压，并使用 chroot 切换工作目录到其中：

```text
# chroot 到目录中
mkdir test
tar xf centos-7-x86_64-docker.tar.xz  -C test/
chroot $PWD/test /bin/bash

# 查看 rootfs 已经切换
ls -l /home/

# 现在直接使用 yum 是不可用的，我们来做一些设置，让 yum 命令可以使用
mount -t proc none /proc
mount -t sysfs none /sys
mount -t tmpfs none /tmp
mount -o bind /dev dev/
mknod -m 666 /dev/random c 1 8
mknod -m 666 /dev/urandom c 1 9
echo "nameserver 114.114.114.114" > /etc/resolv.conf

# 现在执行 yum 已经可以使用了
yum makecache
yum -y install httpd iproute

# 查看安装的软件包
rpm -qa|grep httpd

# chroot 只是改变了 rootfs ，并没有改变其他属性
hostname
ip address show
```

另起一个新的标签页查看变化：

```text
#查到下面的 22966 的进程是由 22965 进程创建
# ps -ef |grep -v grep|grep 22965
root     22965 22441  0 11:20 pts/0    00:00:00 sudo chroot test /bin/bash
root     22966 22965  0 11:20 pts/0    00:00:00 /bin/bash

#查看进程的根目录, 父进程 22965 识别的是 /, 子进程 22966 识别的是 test 目录
# ls -l /proc/2296[5-6]/root
lrwxrwxrwx 1 root root 0 Apr 23 11:50 /proc/22965/root -> /
lrwxrwxrwx 1 root root 0 Apr 23 11:28 /proc/22966/root -> /home/kuops/test

# 刚才安装的 httpd 并没有在主机上安装，只是安装在了 chroot 的 rootfs 中
rpm -qa|grep httpd
```

## Namespace

Linux 的进程都是单一树状结构，所有的进程都是从 `init` 进程开始，通常特权进程可以跟踪或杀死其他的普通进程。Linux 的 NameSpace 将分出多个 `subtree` 子树结构，将进程隔离开来。

```bash
# 通过 /proc/pid/ns 
# ls -l /proc/1/ns
total 0
lrwxrwxrwx 1 root root 0 Apr 23 17:05 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Apr 23 10:55 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 Apr 23 10:55 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Apr 23 10:55 net -> net:[4026532184]
lrwxrwxrwx 1 root root 0 Apr 18 15:07 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Apr 23 10:55 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Apr 23 10:55 uts -> uts:[4026531838]
```

每个进程都具有这 8 个属性

* Cgroup Namespace：让不同的进程组看到不同的 CGroup 规则。
* IPC Namespace：提供基于 system V 进程信道的隔离能力。IPC 全称 Inter-Process Communication， 是 Linux 中的一种标准的进程间通信方式， 包括共享内存，信号量，消息队列等具体方法。
* Mount Namespace： 提供基于磁盘挂载和文件系统的隔离能力。这种隔离效果和 chroot 十分相似，这种效果与 chroot 十分相似，但从实际原理看，mount namespace 会为隔离空间创建独立的 mount 节点树，而 chroot 只改变了当前上下位的根节点 mount 的位置，在文件系统隔离的情况下，无法访问到容器外的任何文件，可以配置挂在额外的目录访问到宿主机的文件系统。
* Network Namespace： 提供基于网络栈的隔离能力，网络栈的隔离允许使用者将特定的网卡与特定容器中的进程进行上下文关联起来，使得同一网卡在主机和容器中分别呈现不同的名称。Network Namespace 的重要作用之一就是让每个容器通过命名空间来隔离和管理自己的网卡配置。因此可以创建一个普通的虚拟网卡，并将它作为特定容器运行环境的默认网卡 eth0 使用。这些虚拟网络网卡最终可以通过某些方式\(NAT,VXLAN,SDN 等\)，连接到实际的物理网卡上，从而实现像普遍主机一样的网络通信。
* PID Namespace： 提供基于进程的隔离能力。进程隔离使得容器中的首个进程成为所在命名空间中的 PID 为 1 的进程。在 Linux 系统中， PID 为 1 的进程非常特殊，它作为所有进程的父进程，有很多特权，如，屏蔽信号，托管孤儿进程等。一个比较直观的现象是，当系统中的某个子进程脱离了父进程（例如父进程意外结束），那么它的父进程就会自动成为系统的根父进程。此外，当系统中的根父进程退出时，所有属于同一命名空间的进程都会被杀死。
* User Namespace：提供基于系统用户的隔离能力。系统用户隔离是指同一系统用户在不同命名空间中拥有不同的 UID （用户标识） 和 GID （组标识）。他们之间存在一定的映射关系。因此在特定的命名空间中 UID 为 0 并不表示该用用拥有整个系统的管理员 root 用户的权限。这一特性限制了容器的用户权限，有利于保护主机系统的安全。
* UTS Namespace： 提供基于主机名的隔离能力。在每个独立的命名空间中，程序都可以有不同的主机名称信息。值得一提的时，主机名只是一个用于标识容器空间的代号，允许重复。

我们使用 unshare 测试一下 uts namespace 的隔离：

```bash
# 测试一下 uts namespace
unshare --uts /bin/bash

# 查看进程名字
# ps -ef|grep /bin/bash
root      9014  5618  0 17:21 pts/0    00:00:00 /bin/bash

# 查看 /proc/pid/ns，会发现 uts ns 是不一样的
ls -l  /proc/9014/ns/
lrwxrwxrwx 1 root root 0 Apr 24 17:21 uts -> uts:[4026532617]

ls -l  /proc/5618/ns/
lrwxrwxrwx 1 root root 0 Apr 24 17:21 uts -> uts:[4026531838]

# 我们在 uts bash 中修改主机名
hostname test.uts.namespace

# 新开窗口查看，在主机中并没有修改 hostname
 hostname
```

测试一下 pid namespace 的隔离：

```bash
unshare --pid --fork --mount-proc /bin/bash

ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  2.3  0.0 119220  7212 pts/0    S    17:41   0:00 /bin/bash
root        37  0.0  0.0 151104  3808 pts/0    R+   17:41   0:00 ps aux
```

测试一下网络隔离：

```bash
unshare --net /bin/bash

ip address show
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop state DOWN qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
```

## CGroup

CGroup 是 Linux 内核提供的一种可以限制，记录，隔离一组进程所使用的物理资源的机制（包括 CPU , 内存，磁盘 I/O 速度）机制。

CGroup 最初设计出来是为了统一 Linux 下资源管理工具，比如限制 CPU 时使用的 `renice` 和 `cpulimit` 命令，限制内存要用 ulimit 或者 PAM \(pluggable Authentication Modules\),而限制磁盘 I/O 又需要其他工具。CGroup 是一种内核级的限制手段，比其他的要的功能和效率方便好得多。CGroup 在 systemd 的 service 文件定义中 `MemoryLimit`,`BlockIOWeight` 等配置其实就是在间接的为进程配置 CGroup。

在 /proc 的文件系统中，每个进程下都有一个 cgroup 文件，与 namespace 类似，觉到多数都一样，共用同一的 namespace：

```bash
# cat /proc/1/cgroup
# 允许或拒绝访问特定设备
12:devices:/
# 限制对内存和 swap 的使用
11:memory:/
# 为进程分配独立的内存
10:cpuset:/
# 为进程限制对块设备的 io
9:blkio:/
# 用于限制能够创建的进程总数
8:pids:/
# 可以挂起或恢复特定的进程
7:freezer:/
# 用于标记每个网络包，并控制网卡优先级
6:net_cls,net_prio:/
# 允许使用 perf 工具来监控 cgroup；
5:perf_event:/
# 限制 RDMA/IB 特定资源，rdma 可以让本机直接存取远程主机的内存资源
4:rdma:/
# 允许使用大篇幅的虚拟内存页，并且给这些内存页强制设定可用资源量。
3:hugetlb:/
# 用于限制进程对 CPU 的用量，并生成每个进程所使用的 CPU 报告。
2:cpu,cpuacct:/
# 由 systemd 创建的这些 cgroup
1:name=systemd:/
```

后面的根是哪里呢，我们看下系统挂载点：

```text
# cat /proc/mounts |grep cgroup
tmpfs /sys/fs/cgroup tmpfs ro,nosuid,nodev,noexec,mode=755 0 0
cgroup /sys/fs/cgroup/systemd cgroup rw,nosuid,nodev,noexec,relatime,xattr,release_agent=/usr/lib/systemd/systemd-cgroups-agent,name=systemd 0 0
cgroup /sys/fs/cgroup/cpu,cpuacct cgroup rw,nosuid,nodev,noexec,relatime,cpu,cpuacct 0 0
cgroup /sys/fs/cgroup/hugetlb cgroup rw,nosuid,nodev,noexec,relatime,hugetlb 0 0
cgroup /sys/fs/cgroup/rdma cgroup rw,nosuid,nodev,noexec,relatime,rdma 0 0
cgroup /sys/fs/cgroup/perf_event cgroup rw,nosuid,nodev,noexec,relatime,perf_event 0 0
cgroup /sys/fs/cgroup/net_cls,net_prio cgroup rw,nosuid,nodev,noexec,relatime,net_cls,net_prio 0 0
cgroup /sys/fs/cgroup/freezer cgroup rw,nosuid,nodev,noexec,relatime,freezer 0 0
cgroup /sys/fs/cgroup/pids cgroup rw,nosuid,nodev,noexec,relatime,pids 0 0
cgroup /sys/fs/cgroup/blkio cgroup rw,nosuid,nodev,noexec,relatime,blkio 0 0
cgroup /sys/fs/cgroup/cpuset cgroup rw,nosuid,nodev,noexec,relatime,cpuset 0 0
cgroup /sys/fs/cgroup/memory cgroup rw,nosuid,nodev,noexec,relatime,memory 0 0
cgroup /sys/fs/cgroup/devices cgroup rw,nosuid,nodev,noexec,relatime,devices 0 0
```

这里每个挂载点都是一个 CGroup 子系统的根目录，例如 `cpuset:/` 对应的根目录其实是 `/sys/fs/cgroup/cpuset`,其他的以此类推。

