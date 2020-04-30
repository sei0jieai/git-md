docker-compose

dockerfile指令笔记

####配置指令

#####ARG
功能：定义创建镜像过程中的变量
格式：ARG <name> [=<default value>]
在执行docker build 时，可以通过-build-args[=]来为变量赋值
 
#####FROM

#####MAINTAINER

#####LABEL

#####EXPOSE

#####ENV

#####ENTRYPOINT

#####VOLUME

#####USER

#####WORKDIR

#####ONBUILD

#####STOPSIGNAL

#####HEALTHCHECK

#####SHELL

####操作指令

#####RUN

#####CMD

#####ADD

#####COPY

###docker-compose yaml指令

####build
指定dockerfile所在路径，可以使绝对路径或相对yaml文件路径，之后Compose会利用它构建并使用该镜像。build可指定创建镜像的上下文，路径等

####cap_add,cap_drop
指定内核能力分配

####command
覆盖容器启动后默认执行的命令

    文件格式示例：
        version: "3"
        services:
          redis:
            image: redis:alpine
            ports:
              - "6379"
            networks:
              - frontend
            deploy:
              replicas: 2
              update_config:
                parallelism: 2
                delay: 10s
              restart_policy:
                condition: on-failure
          db:
            image: postgres:9.4
            volumes:
              - db-data:/var/lib/postgresql/data
            networks:
              - backend
            deploy:
              placement:
                constraints: [node.role == manager]