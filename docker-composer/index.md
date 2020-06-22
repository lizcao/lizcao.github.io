# 什么是docker-composer


### 什么是docker-composer

docker-composer是docker推出的，实现多容器编排的工具

### 相对传统的好处

传统的方式，都是我们会每次docker run一些不同的镜像，如下图

```cpp
//启动nginx
docker run --name nginx -p 80:80 nginx
//启动mysql
docker run --name mysql -p 3306:3306 mysql
```

如果需要不同的镜像之间，能够通讯，还需要写上link的参数，如下图

```cpp
//用link参数来实现nginx对php-fpm的访问
docker run --name nginx -p 80:80 --link php-fpm -d nginx
```

即使是dockerfile的形式，可以在一个容器里面放很多的服务，但是如果想不同的服务容器之间，做分开的管理，还是无法解决问题

所以，docker-composer的方式，就是解决上述的问题的。一次yml文件，即可完成多服务容器的管理

### docker-composer的基本语法

##### image

```cpp
ervices:
  php-fpm:
    image: 'php:fpm'
```

说明: image是指定服务的镜像名称或镜像ID
 tip: 本地不存在，则会做远程拉取

##### build

```cpp
build:
  context: ../
  dockerfile: path/Dockerfile
```

说明:构建dockerfile文件

##### ports

```cpp
services:
  php-fpm:
    image: 'php:fpm'
    //指定端口
    ports:
      - 9000:9000
```

说明:指定不同的端口。即为本机端口映射服务容器端口

##### volumes

```cpp
volumes:
      - ./php/www.conf:/usr/local/etc/php-fpm.d/www.conf
```

说明:挂载文件

##### links

```undefined
links:
      - mysql
```

说明: 链接其他的服务容器

### php环境搭建compose文件

```jsx
version: '2'

services:
  php-fpm:
    image: 'php:fpm'
    ports:
      - 9000:9000
    volumes:
      - ./php/www.conf:/usr/local/etc/php-fpm.d/www.conf
      - ./php/php.ini:/usr/local/etc/php/php.ini
      - ./html:/var/www/html
    links:
      - mysql
    networks:
      - bbc

  nginx:
    image: 'nginx'
    ports:
      - 80:80
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ./html:/usr/share/nginx/html
    links:
      - php-fpm
    networks:
      - bbc

  mysql:
    image: hub.c.163.com/library/mysql:5.6.36
    hostname: mysql
    ports:
      - "127.0.0.1:3306:3306"
    volumes:
      - ./mysql:/var/lib/mysql
    environment:
      - MYSQL_ROOT_PASSWORD=123456
    networks:
      - bbc

networks:
  bbc:
    driver: bridge
```

说明:
 1.总共有3个服务容器，php,nginx,mysql
 2.links做了服务容器的链接
 3.networks的bridge为桥接模式，也是为常见的方式。具体详细的，大家可自行搜索
