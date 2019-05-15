# Ubuntu/Debian

可以启动一个临时镜像跟着练习以下命令:

```text
docker run --rm -it debian:9-slim
```

debian 替换源:

```text
sed -i 's/deb.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list
sed -i 's/security.debian.org/mirrors.aliyun.com/g' /etc/apt/sources.list
```

ubuntu 替换源:

```text
sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
sed -i 's/security.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
```

获取可用软件包列表:

```text
apt-get update
```

升级软件包:

```text
# 升级软件包，忽略内核升级
apt-mark hold linux-image-generic linux-headers-generic
apt-get update
apt-get upgrade -y
```

查看可用软件包:

```text
apt-cache pkgnames
```

模拟安装，`dry-run`:

```text
apt-get install -s nginx
```

安装软件包:

```text
apt-get install nginx -y
```

查看已安装软件包:

```text
apt list --installed
dpkg -l
```

删除软件包:

```text
#保留已修改过的配置文件
apt-get remove nginx
#不保留配置文件
apt-get purge nginx
```

删除非必要软件包:

```text
apt-get autoremove
```

查找软件包:

```text
apt-cache search nginx
```

查看可用软件包:

```text
apt-cache madison nginx
apt-cache policy nginx
```

