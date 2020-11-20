# 目录
- [1.目录结构](#1目录结构)
- [2.快速使用](#2快速使用)
- [3.切换PHP版本](#3切换php版本)
- [4.添加快捷命令](#4添加快捷命令)
- [5.使用Log](#5使用log)
    - [5.1 Nginx日志](#51-nginx日志)
    - [5.2 PHP-FPM日志](#52-php-fpm日志)
    - [5.3 MySQL日志](#53-mysql日志)
- [6.使用composer](#6使用composer)
- [7.数据库管理](#7数据库管理)
    - [7.1 phpMyAdmin](#71-phpmyadmin)
    - [7.2 phpRedisAdmin](#72-phpredisadmin)
- [8.在正式环境中安全使用](#8在正式环境中安全使用)
- [9.常见问题](#9常见问题)
    - [9.1 如何在PHP代码中使用curl？](#91-如何在php代码中使用curl)


## 1.目录结构

```
/
├── conf                    配置文件目录
│   ├── conf.d              Nginx用户站点配置目录
│   ├── nginx.conf          Nginx默认配置文件
│   ├── mysql.cnf           MySQL用户配置文件
│   ├── php-fpm.conf        PHP-FPM配置文件（部分会覆盖php.ini配置）
│   └── php.ini             PHP默认配置文件
├── Dockerfile              PHP镜像构建文件
├── extensions              PHP扩展源码包
├── elasticsearch           ES数据目录
├── log                     Nginx日志目录
├── mysql                   MySQL数据目录
├── redis                   redis数据目录
├── www                     PHP代码目录
└── source.list             Debian源文件
```
结构示意图：
## 2.快速使用
1. 本地安装`git`、`docker`和`docker-compose`。
2. `clone`项目：
    ```
    $ git clone https://github.com/zinopop/scaffold.git
    ```
3. 如果不是`root`用户，还需将当前用户加入`docker`用户组：
    ```
    $ sudo gpasswd -a ${USER} docker
    ```
    _或者请将文件夹权限设置为777 全部角色可读写格式_

4. 如果是纯linux环境下搭建项目，不需要引入docker-sync，请将根目录下的 docker-compose-without-sync.yml 内容拷贝到 docker-compose.yml 中   
    
5. 拷贝环境配置文件`env.sample`为`.env`，启动：
    ```
    $ cd scaffold
    $ cp env.sample .env   # Windows系统请用copy命令，或者用编辑器打开后另存为.env
    $ docker-compose up
    ```
    
6. 访问在浏览器中访问：

 - [http://localhost](http://localhost)： 默认*http*站点
 - [https://localhost](https://localhost)：自定义证书*https*站点，访问时浏览器会有安全提示，忽略提示访问即可

两个站点使用同一PHP代码：`./www/localhost/index.php`。

要修改端口、日志文件位置、以及是否替换source.list文件等，请修改.env文件，然后重新构建：
```bash
$ docker-compose build php56    # 重建单个服务
$ docker-compose build          # 重建全部服务

```



## 3.切换PHP版本
默认情况下，我们同时创建 **PHP5.6和PHP7.2** 三个PHP版本的容器，

切换PHP仅需修改相应站点 Nginx 配置的`fastcgi_pass`选项，

例如，示例的 [http://localhost](http://localhost) 用的是PHP5.6，Nginx 配置：
```
    fastcgi_pass   php56:9000;
```
要改用PHP7.2，修改为：
```
    fastcgi_pass   php72:9000;
```
再 **重启 Nginx** 生效。
```bash
$ docker exec -it dnmp_nginx_1 nginx -s reload
```

## 4.添加快捷命令 （非必须操作）
在开发的时候，我们可能经常使用`docker exec -it`切换到容器中，把常用的做成命令别名是个省事的方法。

打开~/.bashrc，加上：
```bash
alias dnginx='docker exec -it dnmp_nginx_1 /bin/sh'
alias dphp72='docker exec -it dnmp_php72_1 /bin/bash'
alias dphp56='docker exec -it dnmp_php56_1 /bin/bash'
alias dphp54='docker exec -it dnmp_php54_1 /bin/bash'
alias dmysql='docker exec -it dnmp_mysql_1 /bin/bash'
alias dredis='docker exec -it dnmp_redis_1 /bin/bash'
```

## 5.使用Log

Log文件生成的位置依赖于conf下各log配置的值。

### 5.1 Nginx日志
Nginx日志是我们用得最多的日志，所以我们单独放在根目录`log`下。

`log`会目录映射Nginx容器的`/var/log/nginx`目录，所以在Nginx配置文件中，需要输出log的位置，我们需要配置到`/var/log/nginx`目录，如：
```
error_log  /var/log/nginx/nginx.localhost.error.log  warn;
```


### 5.2 PHP-FPM日志
大部分情况下，PHP-FPM的日志都会输出到Nginx的日志中，所以不需要额外配置。

另外，建议直接在PHP中打开错误日志：
```php
error_reporting(E_ALL);
ini_set('error_reporting', 'on');
ini_set('display_errors', 'on');
```

如果确实需要，可按一下步骤开启（在容器中）。

1. 进入容器，创建日志文件并修改权限：
    ```bash
    $ docker exec -it dnmp_php_1 /bin/bash
    $ mkdir /var/log/php
    $ cd /var/log/php
    $ touch php-fpm.error.log
    $ chmod a+w php-fpm.error.log
    ```
2. 主机上打开并修改PHP-FPM的配置文件`conf/php-fpm.conf`，找到如下一行，删除注释，并改值为：
    ```
    php_admin_value[error_log] = /var/log/php/php-fpm.error.log
    ```
3. 重启PHP-FPM容器。

### 5.3 MySQL日志
因为MySQL容器中的MySQL使用的是`mysql`用户启动，它无法自行在`/var/log`下的增加日志文件。所以，我们把MySQL的日志放在与data一样的目录，即项目的`mysql`目录下，对应容器中的`/var/lib/mysql/`目录。
```bash
slow-query-log-file     = /var/lib/mysql/mysql.slow.log
log-error               = /var/lib/mysql/mysql.error.log
```
以上是mysql.conf中的日志文件的配置。

## 6.使用composer
***我们建议在主机HOST中使用composer而不是在容器中使用。***因为：

1. ***composer依赖多。***必须依赖PHP、PHP zlib扩展和git才能使用，PHP和扩展没问题，不过需要安装git，会增大容器体积。

而且需要一个可登陆用户及用户home目录权限。

PHP容器中肯定是安装了

还有就是，在PHP容器中，默认是root，PHP执行的`www-data`用户是没有shell的，也没有home目录。

而且如果有多个PHP版本，每个容器都安装一次composer和git，那是很费时费力费资源的事情。


## 7.数据库管理
本项目默认在`docker-compose.yml`中开启了用于MySQL在线管理的*phpMyAdmin*，以及用于redis在线管理的*phpRedisAdmin*，可以根据需要修改或删除。

### 7.1 phpMyAdmin
phpMyAdmin容器映射到主机的端口地址是：`8080`，所以主机上访问phpMyAdmin的地址是：
```
http://localhost:8080
```

MySQL连接信息：
- host：(本项目的MySQL容器网络)
- port：`3306`
- username： root
- password： 123456

### 7.2 phpRedisAdmin
phpRedisAdmin容器映射到主机的端口地址是：`8081`，所以主机上访问phpMyAdmin的地址是：
```
http://localhost:8081
```

Redis连接信息如下：
- host: (本项目的Redis容器网络)
- port: `6379`


## 8.在正式环境中安全使用
要在正式环境中使用，请：
1. 在php.ini中关闭XDebug调试
2. 增强MySQL数据库访问的安全策略
3. 增强redis访问的安全策略


## 9.常见问题
### 9.1 如何在PHP代码中使用curl？

这里我们使用curl指的是从PHP容器curl到Nginx容器，比如Nginx中我们配置了：
- www.site1.com
- www.site2.com

在site1的PHP代码中，我们要从site1 curl site2服务器，方法如下。

首先，找到Nginx容器的IP地址，命令：
```
$ docker network inspect dnmp_default
...
    "Containers": {
        ...{
            "Name": "nginx",
            ...
            "IPv4Address": "172.27.0.3/16",
            ...
        },
```
这个命令会显示连接到该网络的所有容器，容器nginx的`IPv4Address`就是nginx的IP地址。
修改docker-compose.yml，在php54服务下加上：
```
  php54:
    ...
    extra_hosts:
      - "www.site2.com:172.27.0.3"
```
这样就可以在www.site1.com中curl site2了。

redis 密码可以在.env.sample中找到

在浏览器中输入 
http://127.0.0.1:9200/_cat/health
如果显示为

`1542702332 08:25:32 docker-cluster green 1 1 0 0 0 0 0 0 - 100.0%`

类似样式，证明elasticsearch 安装正确

elasticsearch 会启动两个容器分别为**elasticsearch** ， **elasticsearch2** 方便做实验

官方es docker 搭建url  https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html

## License
MIT


