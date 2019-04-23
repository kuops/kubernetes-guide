# Docker 底层技术

### chroot

在类 Unix 操作系统中，根目录\(root directory\) `/` 是顶级目录，所有文件系统的路径都是从跟 `/` 开始，使用 Chroot 可以更改进程及子进程识别到的根目录 `/` 改为其他目录。

开始一个 chroot 例子,创建一个只有 bash 和 ls 的目录：

```text
mkdir -p test/{bin,lib64}
cp $(which --skip-alias ls) test/bin
cp $(which --skip-alias bash) test/bin
ldd $(which --skip-alias ls)|grep -Po '/lib64/\S+'|xargs -i cp {} test/lib64
ldd $(which --skip-alias bash)|grep -Po '/lib64/\S+'|xargs -i cp {} test/lib64
```

切换到该目录，发现改根目录已经更改：

```text
$ sudo chroot test /bin/bash
# ls /
bin  lib64
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
```

### Namespace

 Linux 的进程都是单一树状结构，所有的进程都是从 `init` 进程开始，通常特权进程可以跟踪或杀死其他的普通进程。Linux 的 NameSpace 将分出多个 `subtree` 子树结构，将进程隔离开来。

```text
# 通过 /proc/pid/ns 
# ls -l /proc/1/ns
total 0
lrwxrwxrwx 1 root root 0 Apr 23 17:05 cgroup -> cgroup:[4026531835]
lrwxrwxrwx 1 root root 0 Apr 23 10:55 ipc -> ipc:[4026531839]
lrwxrwxrwx 1 root root 0 Apr 23 10:55 mnt -> mnt:[4026531840]
lrwxrwxrwx 1 root root 0 Apr 23 10:55 net -> net:[4026532184]
lrwxrwxrwx 1 root root 0 Apr 18 15:07 pid -> pid:[4026531836]
lrwxrwxrwx 1 root root 0 Apr 23 17:05 pid_for_children -> pid:[4026531836](4.12内核以上)
lrwxrwxrwx 1 root root 0 Apr 23 10:55 user -> user:[4026531837]
lrwxrwxrwx 1 root root 0 Apr 23 10:55 uts -> uts:[4026531838]
```

每个进程都具有这 8 个属性，因此每个进程都会分别在这些 Namespace 控制的系统资源上与另一些进程共享空间。在没有使用容器的情况下，系统中所有进程都具有相同的 Namespace ID 组合，如果一个进程运行在 Docker 容器里，他就很可能有一组完全不同的一组 Namespce ID，也可以只做部分隔离，例如使用了 `--network=host`, net Namespace的ID 就会和宿主机的 ID 相同。

* Cgroup Namespace： 提供基于 CGroup （控制组） 的隔离能力。 CGroup 是 Linux 在内核级别对进程可用资源进行限制的一组规则， CGroup 的隔离能让不同的进程组看到不同的 CGroup 规则， 为不同进程组采用各自的配额标准提供便利。
* IPC Namespace：提供基于 system V 进程信道的隔离能力。IPC 全称 Inter-Process Communication， 是 Linux 中的一种标准的进程间通信方式， 包括共享内存，信号量，消息队列等具体方法。IPC 隔离使得只有在同一命名空间下的进程才能相互通信，这一特性对于消除不同容器空间中进程的相互影响具有十分重要的作用。
* Mount Namespace： 提供基于磁盘挂载和文件系统的隔离能力。这种隔离效果和 chroot 十分相似，这种效果与 chroot 十分相似，但从实际原理看，mount namespace 会为隔离空间创建独立的 mount 节点树，而 chroot 只改变了当前上下位的根节点 mount 的位置，在文件系统隔离的情况下，无法访问到容器外的任何文件，可以配置挂在额外的目录访问到宿主机的文件系统。
* Network Namespace： 提供基于网络栈的隔离能力，网络栈的隔离允许使用者将特定的网卡与特定容器中的进程进行上下文关联起来，使得同一网卡在主机和容器中分别呈现不同的名称。Network Namespace 的重要作用之一就是让每个容器通过命名空间来隔离和管理自己的网卡配置。因此可以创建一个普通的虚拟网卡，并将它作为特定容器运行环境的默认网卡 eth0 使用。这些虚拟网络网卡最终可以通过某些方式\(NAT,VXLAN,SDN 等\)，连接到实际的物理网卡上，从而实现像普遍主机一样的网络通信。
* PID Namespace： 提供基于进程的隔离能力。进程隔离使得容器中的首个进程成为所在命名空间中的 PID 为 1 的进程。在 Linux 系统中， PID 为 1 的进程非常特殊，它作为所有进程的父进程，有很多特权，如，屏蔽信号，托管孤儿进程等。一个比较直观的现象是，当系统中的某个子进程脱离了父进程（例如父进程意外结束），那么它的父进程就会自动成为系统的根父进程。此外，当系统中的根父进程退出时，所有属于同一命名空间的进程都会被杀死。
* User Namespace：提供基于系统用户的隔离能力。系统用户隔离是指同一系统用户在不同命名空间中拥有不同的 UID （用户标识） 和 GID （组标识）。他们之间存在一定的映射关系。因此在特定的命名空间中 UID 为 0 并不表示该用用拥有整个系统的管理员 root 用户的权限。这一特性限制了容器的用户权限，有利于保护主机系统的安全。
* UTS Namespace： 提供基于主机名的隔离能力。在每个独立的命名空间中，程序都可以有不同的主机名称信息。值得一提的时，主机名只是一个用于标识容器空间的代号，允许重复。

