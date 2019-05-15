# Dockerfile

多阶段构建示例:

```bash
## git clone code
FROM alpine

ENV MIRROR mirrors.aliyun.com
ENV PACKAGES="git"

WORKDIR /root

RUN sed -i "s/dl-cdn.alpinelinux.org/${MIRROR}/g"  /etc/apk/repositories \
    && apk add --no-cache ${PACKAGES} \
    && git clone https://github.com/katacoda/golang-http-server.git examples

## build app
FROM golang:alpine as builder

ENV MIRROR mirrors.aliyun.com
ENV PACKAGES="git"

WORKDIR /code

COPY --from=0 /root/examples ./examples

RUN sed -i "s/dl-cdn.alpinelinux.org/${MIRROR}/g"  /etc/apk/repositories \
    && apk add --no-cache ${PACKAGES} \
    && cd examples \
    && GOOS=linux go build -o app .

## app image
FROM alpine:latest

WORKDIR /app

COPY --from=builder /code/examples/app .
CMD ["./app"]
```

