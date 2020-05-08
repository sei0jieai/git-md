gitlab迁移文档-20200508-康琳

迁移时间：20200508-23:00

环境：

| 云服务商 | old（阿里云）            | new（华为云）                        |
| -------- | ------------------------ | ------------------------------------ |
| ID       | i-bp1elo0qnx67pggccfo1   | b3b008eb-02c9-40b8-900f-3d340068da59 |
| 配置     | 2核8G                    | 16核64GB                             |
| 公网ip   | 116.62.150.113           | 121.36.58.119                        |
| 私有ip   | 172.16.5.2               | 172.16.1.202                         |
| 带宽     | 5Mbps (峰值)             | 10 Mbit/s                            |
| 到期时间 | 2020年5月10日 23:59 到期 |                                      |

#### 涉及域名：

git.xiaopankeji.com

##### 涉及服务：

- Gitlab
- Jenkins

##### 涉及新服务器端口：

- 9922：ssh端口
- 9980：web访问接口映射docker内部80（宿主机nginx转发）

- 9443：web访问接口映射docker内部443（宿主机nginx转发）

- 18442：tomcat容器8443

### 迁移步骤（docker或者本地安装）：

#### 公用部分：

##### 备份源数据，时间节点截止至20200508-23：00

`gitlab-rake gitlab:backup:create`

##### 备份配置文件，时间节点截止至20200508-23：00

```shell
#nginx——conf
[root@gitlab nginx]# cd /var/opt/gitlab/nginx
[root@gitlab nginx]# tar -zcvf gitlab_nginx_conf_bak.tar.gz conf/
[root@gitlab nginx]# scp -P 20022 gitlab_nginx_conf_bak.tar.gz root@121.36.58.119:/data/


#etc--conf
[root@gitlab nginx]# cd /etc/gitlab/
[root@gitlab gitlab]# vim list1
[root@gitlab gitlab]# cat list1 
gitlab.rb
ssl
trusted-certs
[root@gitlab gitlab]# tar -zcvf gitlab_etc_conf.tar.gz -T list1 
gitlab.rb
ssl/
ssl/git.pem_2017
ssl/git.xiaopankeji.com.zip_bak
ssl/git.pem
ssl/git.xiaopankeji.com.crt_bak
ssl/git.key_2017
ssl/git.key
ssl/git.xiaopankeji.com.crt
ssl/git.xiaopankeji.com.key
ssl/git.xiaopankeji.com.key_bak
trusted-certs/
[root@gitlab nginx]# scp -P 20022 gitlab_etc_conf.tar.gz root@121.36.58.119:/data/
```



##### 发送备份至新服务器

`scp -P 20022 /opt/gitlab_backups/1588946908_2020_05_08_gitlab_backup.tar root@121.36.58.119:/data/`

#### docker：

###### 相关镜像：

**gitlab/gitlab-ce:9.1.3-ce.0**

###### 相关目录（gitlab容器卷目录，迁移请将相关目录一并迁移）：

**/data/gitlab/**

- opt/：对应容器内**/var/opt/gitlab**，gitlab相关数据文件，备份在**opt/backups**

- logs/：对应容器内**/var/log/gitlab**，gitlab相关日志文件

- etc/：对应容器内**/etc/gitlab,gitlab** , 相关配置文件


命令：docker run -d --name gitlab --restart always -v /data/gitlab/etc:/etc/gitlab -v /data/gitlab/logs/:/var/log/gitlab -v /data/gitlab/opt/:/var/opt/gitlab -p 9922:22 -p 9443:443 -p 9980:80 -p 18443:8443 gitlab/gitlab-ce:9.1.3-ce.0

安装服务，创建相关文件夹，检查端口占用

将配置文件备份替换新文件，启动服务

需要替换的文件和文件夹：


etc/:

- /data/gitlab/etc/gitlab.rb                              **#gitlab配置文件**
- /data/gitlab/etc/gitlab-secrets.json

- /data/gitlab/etc/trusted-certs

- /data/gitlab/etc/ssl                                        **#Nginx证书文件夹，新安装没有，需要从old备份过来**


opt/:

- /data/gitlab/opt/nginx/conf/     **#从old备份过来，覆盖**

- /data/gitlab/opt/backups/        **#从old备份过来，迁移的话只需要old上面的最新一份备份**



进入gitlab容器，关闭数据源相关服务，导入备份

`docker exec -it gitlab /bin/bash`

容器内操作：

```bash
gitlab-ctl reconfigure  #重新加载更新的配置文件
gitlab-ctl  stop sidekiq #停止关于数据更新的服务
gitlab-ctl  stop unicorn
gitlab-rake gitlab:backup:restore  BACKUP= force=yes #备份恢复
# 重启gitlab
gitlab-ctl restart
```

##### 修改相关域名信息

![image-20200509001717404](C:\Users\kl\AppData\Roaming\Typora\typora-user-images\image-20200509001717404.png)

测试git与jenkins



```
# git
[root@k8s-master01 ~]# date
Sat May  9 00:17:57 CST 2020
#测试git clone
[root@k8s-master01 ~]# git clone https://root@git.xiaopankeji.com/root/dev-ops-docker-backup.git
Cloning into 'dev-ops-docker-backup'...
Password for 'https://root@git.xiaopankeji.com': 
remote: Counting objects: 28, done.
remote: Compressing objects: 100% (26/26), done.
remote: Total 28 (delta 1), reused 28 (delta 1)
Unpacking objects: 100% (28/28), done.



# 测试git push
[root@k8s-master01 ~]# cd dev-ops-docker-backup/
[root@k8s-master01 dev-ops-docker-backup]# touch test_gitlab_move.txt
[root@k8s-master01 dev-ops-docker-backup]# git add test_gitlab_move.txt 
[root@k8s-master01 dev-ops-docker-backup]# git config --global user.email root@git.xiaopankeji.com
[root@k8s-master01 dev-ops-docker-backup]# git add test_gitlab_move.txt 
[root@k8s-master01 dev-ops-docker-backup]# git commit -m '202005090020_test_gitlab_push'
[master c6e6ecf] 202005090020_test_gitlab_push
 1 file changed, 0 insertions(+), 0 deletions(-)
 create mode 100644 test_gitlab_move.txt
[root@k8s-master01 dev-ops-docker-backup]# git push
warning: push.default is unset; its implicit value is changing in
Git 2.0 from 'matching' to 'simple'. To squelch this message
and maintain the current behavior after the default changes, use:

  git config --global push.default matching

To squelch this message and adopt the new behavior now, use:

  git config --global push.default simple

See 'git help config' and search for 'push.default' for further information.
(the 'simple' mode was introduced in Git 1.7.11. Use the similar mode
'current' instead of 'simple' if you sometimes use older versions of Git)

Password for 'https://root@git.xiaopankeji.com': 
Counting objects: 4, done.
Delta compression using up to 4 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (3/3), 359 bytes | 0 bytes/s, done.
Total 3 (delta 0), reused 0 (delta 0)
To https://root@git.xiaopankeji.com/root/dev-ops-docker-backup.git
   84a5350..c6e6ecf  master -> master
[root@k8s-master01 dev-ops-docker-backup]# 

```

###### push成功

![image-20200509002217826](C:\Users\kl\AppData\Roaming\Typora\typora-user-images\image-20200509002217826.png)



##### Jenkins测试stage环境sync模块

![image-20200509002347560](C:\Users\kl\AppData\Roaming\Typora\typora-user-images\image-20200509002347560.png)



