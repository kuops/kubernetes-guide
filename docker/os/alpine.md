# Alpine

可以启动一个临时镜像跟着练习以下命令:

```text
docker run --rm -it alpine
```

替换源:

```text
sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g'  /etc/apk/repositories
```

安装软件:

```text
apk add --no-cache nginx
```

删除软件:

```text
apk del --no-cache nginx
```

更新本地库:

```text
apk update
```

搜索软件包:

```text
apk search nginx
```

查看所有可用软件包:

```text
apk list
```

查看文件属于哪个包:

```text
apk info --who-owns /etc/passwd
```

查询已安装包的文件:

```text
apk -L info apk-tools
```

软件包是否已安装:

```text
apk -e info
```



