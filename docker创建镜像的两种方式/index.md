# docker创建镜像有两种方式

docker创建镜像有两种方式

## dockerfile方式

### 创建dockerfile

```
FROM php:7.4-cli

ENV DEBIAN_FRONTEND noninteractive
ENV TERM            xterm-color

ARG DEV_MODE
ENV DEV_MODE $DEV_MODE

COPY ./rootfilesystem/ /

RUN \
    curl                      \
        -sfL                  \
        --connect-timeout 5   \
        --max-time         15 \
        --retry            5  \
        --retry-delay      2  \
        --retry-max-time   60 \
        http://getcomposer.org/installer | php -- --install-dir="/usr/bin" --filename=composer && \
    chmod +x "/usr/bin/composer"                                                               && \
    composer self-update 1.10.6                                                                 && \
    apt-get update              && \
    apt-get install -y             \
        inotify-tools              \
        libssl-dev                 \
        supervisor                 \
        unzip                      \
        zlib1g-dev                 \
        --no-install-recommends && \
    install-swoole.sh master && \
    mkdir -p /var/log/supervisor && \
    rm -rf /var/lib/apt/lists/* /usr/bin/qemu-*-static

ENTRYPOINT ["/entrypoint.sh"]
CMD []

WORKDIR "/var/www/"
```

### 根据dockerfile生成镜像

执行命令

```
docker build -t myswool .
```

具体的参数可以参考 

[docker文档build命令]: https://docs.docker.com/engine/reference/commandline/build/


