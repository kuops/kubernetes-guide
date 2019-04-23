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

#查看进程的根目录, 父进程 22965 识别的是 /, 子进程识别的是 test 目录
# ls -l /proc/2296[5-6]/root
lrwxrwxrwx 1 root root 0 Apr 23 11:50 /proc/22965/root -> /
lrwxrwxrwx 1 root root 0 Apr 23 11:28 /proc/22966/root -> /home/kuops/test
```

