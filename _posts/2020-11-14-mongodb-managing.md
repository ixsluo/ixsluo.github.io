---
layout: post
title: MongoDB Managing
subtitle: 
author: ixsluo
date: 2020-11-14 21:36:40 +0800
categories: 
tag: 
---

## 容器中安装MongoDB

拉取最新版本镜像
```sh
docker pull mongo:latest
```

运行镜像mongo，容器名mongo，启用mongo密码
```sh
docker run -itd -p 27017:27017 --name mongo mongo --auth
```

进入MongoDB，并使用admin数据库
```sh
docker exec -it mongo mongo admin
```

设置管理账户admin
```txt
> db.createUser({
... user: "admin",
... pwd: "******",
... roles: [{
...     role: "userAdminAnyDatabase",
...     db: "admin"}],
... })

验证连接
> db.auth('admin', '******')
```

## 备份与恢复容器内的MongoDB


容器与宿主机文件拷贝，容器内路径需加<容器名:>
```bash
docker cp <hostdir> <container>:<dir>
```

容器内进行备份与恢复
```bash
docker exec -it mongo /bin/bash  # 进入容器
mongodump -d <db> -o <dir>  # 将db数据库备份至dir目录

mongorestore -d <db> <path>  # 恢复数据库，-d重命名。<path> <- <dir>/db
mongorestore --dir <dir>  # 恢复整个备份目录
```
