---
categories: [tool]
tags: [windows, docker]
---

> 每次重装系统后，都要装MySQL，这是一个重复且无趣的动作。正好前段时间windows家庭版也支持了docker。所以，这次重装系统后我决定用docker来装MySQL。设置好对应的配置项然后上传到[dockerhub](https://hub.docker.com/)，一劳永逸。


windows下安装docker可以参考[这篇文章](https://www.yuque.com/yigenranshaodexiongmao/fgx0oh/kng76e)，这里不再赘述。

## 拉取MySQL镜像

```css
# 拉取指定版本镜像
docker pull mysql:version
# 不指定版本默认拉取最新的版本（我选的这个）
docker pull mysql
```

镜像下载完后，查看镜像信息

```bash
docker images
```

我这里下载的镜像信息如下：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2391485/1636138207742-f8aa7111-be70-46da-be23-9e0d25a08608.png#clientId=u4ed3618e-460c-4&from=paste&height=242&id=u756a12f3&originHeight=242&originWidth=621&originalType=binary&ratio=1&size=20878&status=done&style=none&taskId=u83dd2ab5-90db-425d-80d9-40a852062e7&width=621)


然后就是启动镜像了。由于container删除后，MySQL的数据也会随之消失，这是我们接受不了的，所以我们需要挂载一下配置文件和数据，将其存储在物理机上，启动命令如下：

```bash
docker run --name mysql8.0 -p 8000:3306  
-v D:/Work/DockerVolumes/mysql/data/:/var/lib/mysql 
-v D:/Work/DockerVolumes/mysql/my.cnf:/etc/mysql/conf.d/my.cnf 
-v D:/Work/DockerVolumes/mysql/log:/var/lib/log 
-v D:/Work/DockerVolumes/mysql/mysql-files:/var/lib/mysql-files 
-e MYSQL_ROOT_PASSWORD=123456 mysql
```

这条命令做了以下几个事情：

- 启动镜像，将容器命名为mysql8.0
- -p：将物理机端口8000映射到MySQL的端口
- -v：将MySQL的配置文件、日志和数据分别挂载到物理机的目录
- 设置MySQL的密码为123456

值得一提的是，MySQL支持多个位置的my.cnf的加载。它默认的加载顺序是/etc/my.cnf、/etc/mysql/my.cnf，最后是用户自定义位置的~/.my.cnf。

因为默认的my.cnf配置文件中有这么一句话

```bash
# Custom config should go here
!includedir /etc/mysql/conf.d/
```

所以，我这里选择了/etc/mysql/conf.d/my.cnf当我的挂载目录。

我的my.cnf配置如下
```bash
# Copyright (c) 2017, Oracle and/or its affiliates. All rights reserved.
#
# This program is free software; you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation; version 2 of the License.
#
# This program is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with this program; if not, write to the Free Software
# Foundation, Inc., 51 Franklin St, Fifth Floor, Boston, MA  02110-1301 USA

#
# The MySQL  Server configuration file.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

[mysqld]
pid-file        = /var/run/mysqld/mysqld.pid
socket          = /var/run/mysqld/mysqld.sock
datadir         = /var/lib/mysql
```

启动成功后，进入容器：

```bash
 docker exec -it mysql8.0 /bin/bash
```

连接MySQL:

```bash
mysql -uroot -p123456
```

至此，第一部分成功。

## 通过Navicat连接MySQL

MySQL默认不允许远程连接，所以我们需要先修改权限：

1.  允许root用户远程连接MySQL数据库  
```bash
grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option;
```

2.  刷新权限  
```bash
flush privileges;
```

使用Navicat连接成功

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2391485/1636138233340-3bb37d29-cd79-40c4-b33f-94d68b804254.png#clientId=u4ed3618e-460c-4&from=paste&height=462&id=u5ed7c8d7&originHeight=462&originWidth=535&originalType=binary&ratio=1&size=37634&status=done&style=none&taskId=u930d98f4-9c08-4669-930c-1d8e0771e03&width=535)

## 验证挂载配置是否生效

### 验证my.cnf配置是否生效

慢查询功能默认是关闭的：

```bash
show variables like '%slow_query%';
```

结果如下：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2391485/1636563692161-ed995295-699f-40e1-b0c6-54038bf6dbcc.png#clientId=ub96e022c-7e07-4&from=paste&height=154&id=u45b8495e&originHeight=154&originWidth=585&originalType=binary&ratio=1&size=11894&status=done&style=none&taskId=u5ca23671-55af-43bf-b3cf-fed59ee4404&width=585)

然后我们修改D:/Wor/DockerVolumes/mysql/my.cnf文件

```csharp
[mysqld]
# 是否开启慢查询日志
slow_query_log=1
# 慢查询的阈值, 单位秒
long_query_time=0
# 慢日志输出位置
slow_query_log_file=/var/lib/log/slow.log
```

这里你可以直接在你的物理机上修改，但是这时你的docker容器中相关挂载的文件的权限会变成777。在MySQL8.0中，高权限的配置文件不会生效，所以我们需要修改权限：

```bash
chmod 644 /etc/mysql/conf.d/mysql.cnf
```

然后退出docker，重启容器：

```basj
docker restart continerId
```

查看效果，已经生效：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2391485/1636563792785-edfc0e66-6771-457e-a984-11c769b962f6.png#clientId=ub96e022c-7e07-4&from=paste&height=163&id=u0dd564d9&originHeight=163&originWidth=448&originalType=binary&ratio=1&size=10332&status=done&style=none&taskId=ue85733ef-7624-4504-8a9a-72bf56dc4df&width=448)
因为我们挂载了日志的输出位置，现在查看D:\Work\DockerVolumes\mysql\log下生成的slow.log:
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2391485/1636563875103-cc6adaba-f477-4437-b9b1-0fc573fdfd9d.png#clientId=ub96e022c-7e07-4&from=paste&height=787&id=uce8fa4b8&originHeight=787&originWidth=1195&originalType=binary&ratio=1&size=106132&status=done&style=none&taskId=u1ed52ada-29d0-4bbd-9a26-7182d0478f9&width=1195)
**日志目录挂在成功**

### 验证数据挂载目录是否生效

在navicat中新建test数据库并新建表t，字段t并新增一条t=1的数据：
![image.png](https://cdn.nlark.com/yuque/0/2021/png/2391485/1636138257222-ef166fa0-909a-4f7e-9c72-2fefbad61d55.png#clientId=u4ed3618e-460c-4&from=paste&height=177&id=ud8cc5adb&originHeight=177&originWidth=846&originalType=binary&ratio=1&size=17391&status=done&style=none&taskId=uc2d4367c-e99d-4fc8-9bde-5dda0c284fe&width=846)
	

创建完之后，我们查看下挂载目录：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2391485/1636138267062-17d83f57-a917-4bee-be20-9644070044c8.png#clientId=u4ed3618e-460c-4&from=paste&height=595&id=u55232d1a&originHeight=595&originWidth=794&originalType=binary&ratio=1&size=46769&status=done&style=none&taskId=u99a958e9-e296-40e2-8835-d8f06c921f6&width=794)

进入test查看：

![image.png](https://cdn.nlark.com/yuque/0/2021/png/2391485/1636138272714-5e3aa4f1-6eaa-4ceb-854c-b8bebf042ba6.png#clientId=u4ed3618e-460c-4&from=paste&height=183&id=ub66a4a00&originHeight=183&originWidth=735&originalType=binary&ratio=1&size=11164&status=done&style=none&taskId=u702c887a-f932-47f2-84ab-ae1e9ca595b&width=735)

表t的数据文件已存在，挂在成功

## 常见问题

1.  mysqld: Error on realpath() on '/var/lib/mysql-files' (Error 2 - No such file or directory)  
> 需要挂载mysql-files目录：-v D:/Work/DockerVolumes/mysql/mysql-files:/var/lib/mysql-files。

2.  World-writable config file '/etc/mysql/my.cnf' is ignored  
> MySQL8.0中，当my.cnf权限为777时，该配置文件将不会生效，所以需要修改权限：chmod 644 /etc/mysql/my.cnf后，重启容器才会读取我们挂载的配置文件。


## 总结

其实这个操作十分简单，核心是docker run的启动命令配置。使用我提供的命令可以快速启动一个MySQL8.0的容器，节省了很多的物理机安装及配置时间，方便快捷，赶紧试试吧！
