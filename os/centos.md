# CentOS

可以启动一个临时镜像跟着练习以下命令:

```text
docker run --rm -it centos:7
```

虽然  centos 镜像中自带了 fastestmirror 插件，但是一般还是会选择到国外，这里使用阿里云镜像，并添加 epel 源:

```text
curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
curl -o /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
yum clean all
yum makecache
```

升级软件包:

```text
yum update --exclude=kernel*
```

查看可用软件包:

```text
yum list
```

安装软件包:

```text
yum -y install nginx
```

查看软件包信息:

```text
yum info nginx
```

查看可用版本,并安装指定版本:

```text
yum --showduplicates list docker
yum -y install docker-2:1.13.1-94.gitb2f74b2.el7.centos
```

查看已安装版本:

```text
yum list installed
```

降级版本:

```text
yum downgrade python-libs python
```

重装软件:

```text
yum reinstall nginx
```

卸载软件:

```text
yum remove nginx
yum erase nginx
```

删除不是必须的软件

```text
yum autoremove
```

查询文件属于哪个包:

```text
yum provides /etc/passwd
```

yum 安装时启用禁用仓库

```text
yum --disablerepo=epel install httpd
yum --enablerepo=epel install nginx
yum --disablerepo=* --enablerepo=epel install httpd
```

