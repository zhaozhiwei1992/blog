---
title: docker实践
date: 2019-07-19
tags: docker, linux
---

环境准备
========

-   华为云 centos系统
-   本地已经正常使用, 需要部署到服务器并且需要用docker启动
-   web容器使用weblogic10.3.6
-   本人的原始环境为archlinux

操作步骤
========

本地环境打包
------------

-   注意： 采用本地环境为了省事, 本地环境包含整个的weblogic环境， 及jdk,
    如果自己心情好完全可以再搭建一套

``` {.example}
tar -zcvf weblogic.tar.gz ~/Oracle #这个目录是weblogic的默认安装及建域目录, 相应的war包也在该目录下
tar -zcvf jdk.tar.gz ~/applications/jdk1.7 #jdk
```

上传到云服务器
--------------

-   上传打包好的文件
    -   weblogic

        ``` {.example}
        scp weblogic.tar.gz root@11x.xxx.xxx.xxx:/root  
        根据提示输入密码稍等上传完成
        ```

    -   jdk

        ``` {.example}
        scp jdk.tar.gz root@11x.xxx.xxx.xxx:/root  
        根据提示输入密码稍等上传完成
        ```

-   解压, 找一个自己喜欢的目录，建议跟本地保持一致
    -   在服务建立与本地一致的目录

        ``` {.example}
        mkdir ~/applications
        ```

    -   解压文件

        ``` {.example}
        tar -zxvf /root/weblogic.tar.gz -C ~
        tar -zxvf /root/jdk.tar.gz -C ~/applications
        ```

-   在云服务器上配置启动web服务,
    并通过弹性公网访问或者服务器本地确认服务启动正常
    -   进入domain目录启动服务验证

        ``` {.example}
        nohup ./startWeblogic.sh & tail -f nohup.out
        ```

使用docker容器启动
------------------

### 制作docker镜像

-   Dockerfile文件

    -   touch ~/Dockerfile

        ``` {.example}
        #Dockerfile  
        # 基于centos构建本容器
        FROM centos
        # 跟本地映射的端口， 这里相当与访问本地7003会自动转发到容器中7003
        EXPOSE 7003
        #设置环境变量，所有操作都是非交互式的
        ENV LANG en_US.UTF-8  
        ENV LC_ALL en_US.UTF-8
        ENV DEBIAN_FRONTEND noninteractive
        #可以安装软件
        #RUN yum install xx
        # 需要打到docker中的文件全部copy到docker对应的目录中， 本地目录是相对当前目录，不能写绝对路径, 后面这个路径最好加/
        COPY . ~/
        COPY cmd.sh /cmd.sh
        # 容器启动后直接执行给定脚本
        CMD sh /cmd.sh
        #End
        ```

    -   制作镜像

        -   进入目录 ~ 在目录 ~中目前存在

            ``` {.example}
            drwxr-xr-x 3 root root 4096 Sep 30 10:49 applications
            -rwxr-xr-x 1 root root  128 Sep 30 11:17 cmd.sh
            -rw-r--r-- 1 root root  305 Sep 30 17:07 Dockerfile
            drwxr-xr-x 3 root root 4096 Jul 14 18:39 Oracle
            ```

        -   执行命令 docker build -t payimage .

            -   最后的点表示dockerfile在当前目录

        -   查看存在镜像 docker images

            ``` {.example}
            REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
            payimage            latest              d7b3db0e1d59        15 hours ago        2.22 GB
            centos              latest              c5507be714a7        7 weeks ago         199.7 MB
            ```

-   docker commit命令通过运行中容器制作镜像

    ``` {.example}
    docker commit pay7003 payimages:v1.0
    ```

### 启动容器

-   cmd.sh

    ``` {.example}
    #!/bin/bash
    domainname=$1 # 这里如果没有其他情况可以写死
    nohup sh ~/Oracle/Middleware/user_projects/domains/$domainname/startWebLogic.sh > /7003.log & tail -f /7003.log 
    ```

-   通过上一步镜像启动

    ``` {.example}
    docker  run -itd --net=host --name pay7003 payimage sh /cmd.sh
    命令解释:
    -i: 
    -t: 
    -d: 保持，退出后不关闭docker
    --name: docker容器的名称
    payimage: 镜像名称
    --net=host: 这个必须有，否则启动报错
    sh /cmd.sh: 这个是容器启动会执行的命令， 以后docker start 也会执行该文件
    ```

-   可以省略第一步操作直接制作容器启动, 不使用docfile直接制作镜像

    ``` {.example}
    docker run itd -v weblogic:weblogic idk:jdk --net=host -p7003:7003 --name=pay7003 centos
    上面的weblogic前后分别表示本地的目录和远程的目录
    -v 可以达到dockfile中copy的效果
    -p 表示端口映射
    ```

-   容器管理

    ``` {.example}
    docker start pay7003
    docker stop pay7003
    docker rm pay7003 #删除容器, 先stop再rm
    ```

### 连接容器，检查服务

-   查看现有容器

    -   docker ps -a

        ``` {.example}
        CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
        dc2eeca7873e payimage "sh /cmd.sh" 15 hours ago Exited (137) 15 hours ago pay7003 
        ```

-   登录容器

    ``` {.example}
    docker exec -it pay7003 bash #这里pay7003是容器name， 也可以使用容器id
    通过该命令可以连接到容器pay7003的bash中， 跟正常操作cenos系统一样, 完成后exit即可
    ```

容器管理
========

大杀器片甲不留, 自己玩可以用
----------------------------

``` {.example}
docker kill $(docker ps -q) ; docker rm $(docker ps -a -q) ; docker rmi $(docker images -q -a) 
```

删除镜像
--------

``` {.example}
docker rmi payimage
```

问题处理
========

archlinux中启动docker实例报错
-----------------------------

-   systemctl start docker.service报错，直接使用systemctl enable
    docker.service重启机器正常

docker镜像持久化到本地(待测试)
------------------------------

-   有的项目太大了，而且是公司内部使用不合适到公网
-   docker save java -o ./java.tar
-   copy到其他伙伴机器后可以使用: docker load -i java.tar,
    加载到对方容器中
-   参考: <https://segmentfault.com/a/1190000006843830>

docker到底能干个啥?
-------------------

-   在团队内部构建本地的仓库，标准化所有的开发环境，使得团队的新人可以快速上手
-   在生产环境部署Docker，这其实是PaaS的虚拟化和自动化的一种方式，利用LXC和Docker能够更便捷地实施PaaS
-   尝试用Docker做分布式集群模拟和测试，成本会更加低廉，更加容易维护
-   参考:
    <https://www.csdn.net/article/2014-06-19/2820312-Docker-lxc-paas-virtualization>
