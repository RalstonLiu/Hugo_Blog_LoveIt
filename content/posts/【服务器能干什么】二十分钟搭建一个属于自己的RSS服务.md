---
title: "【服务器能干什么】二十分钟搭建一个属于自己的RSS服务"
date: 2020-10-05T16:10:20+08:00
draft: false
                   
description : "这是描述信息"           
lastmod: 2020-10-06             

tags : [                                    
"Docker","TinyTinyRss",
]
categories : [                              
"技术向",
]
keywords : [                                
"Docker",
]

---

## 前言
### RSS 服务


市面上有非常多的 RSS 聚合服务，来帮助我们统一管理、订阅、更新、筛选 RSS 源推送给我们的更新信息，避免我们被海量的文章淹没，也能保证我们多个设备上 RSS 的阅读进度一致。

Feedly、Inoreader 等等都是非常不错的 RSS 服务，但是它们的免费版本都有着一定的限制，有时候无法满足我们的全部功能需求，而动辄一个月数十刀的订阅费用又让人望而却步。

不过，GitHub上有一个开源的 RSS 服务：Tiny Tiny RSS

可以满足我们 RSS 订阅的全部需求！

准备工作：一台服务器（vps）

## 1、安装docker
Docker 是非常优秀的虚拟化容器，借助于 Docker 我们可以方便的部署 Tiny Tiny RSS，首先我们在服务器上安装 Docker 本体。在服务器上面执行下面命令来安装 Docker：

```bash
curl -fsSL https://get.docker.com/ | sh
```
然后启动 Docker 服务：
```bash
sudo systemctl start docker
```
然后，我们检查一下 Docker 是否启动成功。我们执行命令 `sudo systemctl status docker`:


![启动成功](https://img-blog.csdnimg.cn/20201002222919937.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)


## 2、安装docker-compose
接下来我们安装 docker-compose：一个管理和启动多个 Docker 容器的工具。

由于 Tiny Tiny RSS 依赖有 PostgreSQL 的数据库服务以及 mercury_fulltext 的全文抓取服务等等，这些服务我们都借助于 Docker 部署，因此利用 docker-compose 就会大大降低我们的部署难度。

我们继续，在服务器上面执行下面的命令来安装 docker-compose：
```bash
curl -L https://github.com/docker/compose/releases/download/1.25.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
```
之后给予安装好的 `docker-compose` 可执行权限：
```bash
chmod +x /usr/local/bin/docker-compose
```
最后我们运行 `docker-compose --version` 来检查安装是否成功。如果有如下输出，说明我们的 `docker-compose` 安装成功：

![安装成功](https://img-blog.csdnimg.cn/20201002223145515.png#pic_center)

## 3、安装Tiny Tiny RSS
准备工作已经全部完成，接下来我们下载由 [Awesome-TTRSS](https://ttrss.henry.wang/zh/#%E5%85%B3%E4%BA%8E) 配置的 Tiny Tiny RSS 服务的 `docker-compose` 配置文件：

```bash
# 创建 ttrss 目录并进入
mkdir ttrss && cd ttrss

# 利用 curl 下载 ttrss 的 docker-compose 配置文件至服务器
curl -fLo docker-compose.yml https://github.com/HenryQW/Awesome-TTRSS/raw/master/docker-compose.yml
```

修改 `docker-compose.yml `里面的内容：


- 在配置文件的第 7 行和第 23 行，将 PostgreSQL 数据库的默认密码进行修改。暴露在公网的数据库使用默认密码非常危险。

- 在配置文件的第 18 行，将 Tiny Tiny RSS 服务的部署网址修改为我们实际的部署网址。比如我的部署网址是 :`https://rss.yirenliu.cn`

- 注意，如果你的部署 URL 包含端口（默认部署端口为 181 端口），那么这里的 URL 也需要加上端口号，格式为 {网址}:{端口}不过不必担心，如果你这里的 URL 配置不正确，那么访问 Tiny Tiny RSS 的时候，Tiny Tiny RSS 会提醒你修改这里的值为正确的 URL，按照提醒进行配置即可

贴一个我的配置文件给大家参考：

```yml
version: "3"
services:
  database.postgres:
    image: postgres:13-alpine
    container_name: postgres
    environment:
      - POSTGRES_PASSWORD=passwd # please change the password
    volumes:
      - ~/postgres/data/:/var/lib/postgresql/data # persist postgres data to ~/postgres/data/ on the host
    restart: always

  service.rss:
    image: wangqiru/ttrss:latest
    container_name: ttrss
    ports:
      - 8002:80
    environment:
      - SELF_URL_PATH=https://rss.yirenliu.cn # please change to your own domain
      - DB_HOST=database.postgres
      - DB_PORT=5432
      - DB_NAME=ttrss
      - DB_USER=postgres
      - DB_PASS=passwd # please change the password
      - ENABLE_PLUGINS=auth_internal,fever # auth_internal is required. Plugins enabled here will be enabled for all users as system plugins
      - FEED_LOG_QUIET=true
    stdin_open: true
    tty: true
    restart: always
    command: sh -c 'sh /wait-for.sh $$DB_HOST:$$DB_PORT -- php /configure-db.php && exec s6-svscan /etc/s6/'

  service.mercury: # set Mercury Parser API endpoint to `service.mercury:3000` on TTRSS plugin setting page
    image: wangqiru/mercury-parser-api:latest
    container_name: mercury
    expose:
      - 3000
    restart: always

  service.opencc: # set OpenCC API endpoint to `service.opencc:3000` on TTRSS plugin setting page
    image: wangqiru/opencc-api-server:latest
    container_name: opencc
    environment:
      - NODE_ENV=production
    expose:
      - 3000
    restart: always

  # utility.watchtower:
  #   container_name: watchtower
  #   image: containrrr/watchtower:latest
  #   volumes:
  #     - /var/run/docker.sock:/var/run/docker.sock
  #   environment:
  #     - WATCHTOWER_CLEANUP=true
  #     - WATCHTOWER_POLL_INTERVAL=86400
  #   restart: always
```

上面是完整的配置结果，我把端口改成了8002，如果是用宝塔面板，记得打开`安全`里面的`端口设置`:

![开启对应的端口](https://img-blog.csdnimg.cn/20201002223922201.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

我这边是腾讯云轻量应用服务器，还需要在在防火墙打开8002端口：

![打开相应的端口](https://img-blog.csdnimg.cn/20201002224226853.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

之后，我们保存配置文件，启动 Tiny Tiny RSS 服务。在刚刚的 ttrss 目录下执行：
```bash
docker-compose up -d
```

等待脚本执行完成，如果一切没有问题，那么接下来输入 `docker ps`，我们应该看到类似下面的结果：

![最后一个是我的博客 - -！](https://img-blog.csdnimg.cn/20201002224348402.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

上面内容表示我们开启了四个 Docker 容器，分别是：

- Tiny Tiny RSS 本身，监听端口为 0.0.0.0:8002 → 80，同时暴露给外网
- PostgreSQL 数据库服务
- Mercury 全文抓取服务
- OpenCC 简体、繁体中文转换服务

如果发现问题，修改 docker-compose 的配置文件后，需要执行下面的命令重启 Docker 容器们：

```bash


# 关闭 Docker 容器们
docker-compose down

# 删除已停止的 Docker 容器
docker-compose rm

# ……
# 修改 docker-compose 配置文件
# ……

# 再次开启 Docker 服务
docker-compose up -d

```






## 4、配置nginx反向代理

事实上，到上一步，如果我访问 {服务器 IP}:8002，应该可以直接看到 Tiny Tiny RSS 的 Web 前端，但是 Tiny Tiny RSS 并不能直接配置 SSL 证书，也就没法添加 HTTPS 支持。

我们可以利用 Nginx 作为反向代理服务器，即可方便的给 Tiny Tiny RSS 单独绑定一个我们希望的域名，并利用[ Let's Encrypt](https://letsencrypt.org/) 来部署 HTTPS。


还记得我在之前的`docker-compose.yml`里面，我填好的地址是`https://rss.yirenliu.cn`


这里，我在宝塔面板新建一个站点：

![不要数据库，不要php，纯静态即可](https://img-blog.csdnimg.cn/2020100222490215.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

配置好SSL证书：

![免费证书Let's Encrypt，宝塔会自动续期](https://img-blog.csdnimg.cn/20201002225011367.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

修改后部分的配置文件：


![注意观察](https://img-blog.csdnimg.cn/20201002225415603.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)



```nginx
# location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
    # {
    #     expires      30d;
    #     error_log off;
    #     access_log /dev/null;
    # }
    
    # location ~ .*\.(js|css)?$
    # {
    #     expires      12h;
    #     error_log off;
    #     access_log /dev/null; 
    # }
                location / {
    proxy_pass http://127.0.0.1:8002/;
    rewrite ^/(.*)$ /$1 break;
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Upgrade-Insecure-Requests 1;
    proxy_set_header X-Forwarded-Proto https;
  }
    access_log  /www/wwwlogs/rss.yirenliu.cn.log;
    error_log  /www/wwwlogs/rss.yirenliu.cn.error.log;
}
```



主要是注释了52-64的内容：

```bash
    # location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
    # {
    #     expires      30d;
    #     error_log off;
    #     access_log /dev/null;
    # }
    
    # location ~ .*\.(js|css)?$
    # {
    #     expires      12h;
    #     error_log off;
    #     access_log /dev/null; 
    # }


```


添加了65-75行的内容，


```bash

location / {
    proxy_pass http://127.0.0.1:8002/;
    rewrite ^/(.*)$ /$1 break;
    proxy_redirect off;
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Upgrade-Insecure-Requests 1;
    proxy_set_header X-Forwarded-Proto https;
  }

```

端口改成你自己在`docker-compose.yml`设置的端口即可。



注意：这个方法可以解决宝塔面板默认只能配置一个反向代理的情况！！！


![不要用宝塔自带的反向代理设置](https://img-blog.csdnimg.cn/20201002225722149.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

> 用宝塔自带的反向代理设置，会出现只能设置一个网站反向代理的情况，就是说你设置了这个，如果你之后还想用反向代理，再在这边设置，会出错！！！亲测！！！



## 5、登陆配置全文阅读与繁简体转换
接下来就能通过域名正常访问了。

![登陆界面](https://img-blog.csdnimg.cn/20201002230017186.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

默认账户：`admin`

密码：`password`


第一次登陆，务必修改一下默认的密码:

![右上角进入偏好设置](https://img-blog.csdnimg.cn/20201002230411676.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)



如果上面步骤没有问题的话，我们在服务器上面所部署的 Tiny Tiny RSS 本身就已经包含了：

- Mercury 全文提取服务（默认未开启）
- OpenCC 繁简自动转换服务（默认未开启）
- Fever 格式输出插件（默认已开启，用来和 Reeder 等客户端进行连接）
- 包括 Feedly、RSSHub 在内的多款主题
- 等等……

我们不需要多余的配置，开箱即可使用上面的主题和插件，根本不需要操心其他服务的部署和安装。我们登录自己的 Tiny Tiny RSS，在右上角「设置→ 插件」中即可启用上述插件，在「设置 → 主题」处就可以更改我们部署的 Tiny Tiny RSS 所用的主题。

如果有同学对上面的配置还有问题，请直接参考 Awesome TTRSS 的官方文档：[🐋 Awesome TTRSS | 插件](https://ttrss.henry.wang/zh/#%E6%8F%92%E4%BB%B6)



## 6、如何在多平台阅读


![启用API](https://img-blog.csdnimg.cn/20201002230458907.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

这个API务必开启，后续我们会在电脑和手机上安装软件来阅读内容，毕竟现在是手机的时代，只在浏览器上看没啥软用。


然后配置一下密码：

![注意这个地址，后续我们会用到](https://img-blog.csdnimg.cn/20201002231834836.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)
然后启用一下插件：

![mercury_fulltext全文阅读插件开启](https://img-blog.csdnimg.cn/20201002230718532.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)


![繁简体转换插件开启](https://img-blog.csdnimg.cn/20201002230800117.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)


配置全文阅读插件，输入`service.mercury:3000`：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201002230856543.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)
配置繁简转换插件，同样输入`service.mercury:3000`：

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201002230926374.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

## 7、享受自己的信息流

![添加信息分类](https://img-blog.csdnimg.cn/20201002231042403.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)
订阅信息源：

![订阅信息源](https://img-blog.csdnimg.cn/20201002231119131.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)
这里推荐一个神级项目：[RSSHub](https://docs.rsshub.app/usage.html#sheng-cheng-ding-yue-yuan)

在这里你能找到一切rss源，具体可以自己看一下，这边就不展开了。

![我们不需要自己编写订阅源，这边点进去自己找](https://img-blog.csdnimg.cn/20201002232933348.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

![这个就是RSS的订阅地址](https://img-blog.csdnimg.cn/20201002233025150.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)



一般的话，你在网站的域名最后加上`atom.xml`,有大概率就是对应的rss地址了，比如我的博客域名是`https://yirenliu.cn/`,我的rss地址就是`https://yirenliu.cn/atom.xml`

![就是这样滴，密密麻麻](https://img-blog.csdnimg.cn/20201002231459392.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

我这边用的是MAC,在appstore上面下载这个：

![reeder](https://img-blog.csdnimg.cn/20201002233241630.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)



![reader](https://img-blog.csdnimg.cn/2020100223164020.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

![这个地方填三个东西](https://img-blog.csdnimg.cn/20201002232319605.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

`server`的话填前面我们的那个地址,我的是：`https://rss.yirenliu.cn/plugins/fever/`；

`email` 这边是个坑，我搞了好久，最后其实是填的用户名，我们默认的填`admin`就好；

`password`密码的话就填前面那个API设置过的密码就好。

然后就能成功登陆了。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20201002232245954.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

IOS上也是这个软件，不过貌似要付钱，我花了好像30块钱买的。

![就是这个](https://img-blog.csdnimg.cn/20201002232754692.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)


![我的部分订阅内容](https://img-blog.csdnimg.cn/20201002232842791.jpeg?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

![MAC上的效果](https://img-blog.csdnimg.cn/20201002233133202.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

![网页端效果](https://img-blog.csdnimg.cn/20201002233530202.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)

![早睡早起，已经快晚上十二点了......](https://img-blog.csdnimg.cn/20201002233707618.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1JSaWRkbGU=,size_16,color_FFFFFF,t_70#pic_center)


订阅自己感兴趣的内容，拒绝被所谓的智能算法强行推荐内容，有了这个RSS服务，我们可以把阅读信息的主动权掌握在自己的手中。
