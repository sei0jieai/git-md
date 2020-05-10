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





#添加定时任务，进行定时备份，以及删除七天以前的备份

三种方式：

1. 宿主机，
2. 创建单独cron容器https://hub.docker.com/r/willfarrell/crontab，未试验
3. 容器内

###### 添加脚本，放在容器内的/var/opt/gitlab/myshell/gitlab_bak.sh,容器需要创建一个日志文件夹/var/log/gitlab/myshell/，日志放在/var/log/gitlab/myshell/logs，

``````
#!/bin/bash

Shell_Name=gitlab_bak.sh
Log_Path=/var/log/gitlab/myshell/logs
Log_File=${Log_Path}/${Shell_Name}.log
Gitbak_Path=/var/opt/gitlab/backups

function Shell_log()
{
    echo "$(date +"%Y-%m-%d") $(date +"%H:%M:%S") ${Shell_Name} $1" >> ${Log_File}
}

function Main(){
    if [ ! -d ${Log_Path} ];then
        mkdir -p ${Log_Path}
    fi
    Shell_log "Shell start..."

    /usr/bin/gitlab-rake gitlab:backup:create
    if [ $? -eq 0 ];then
        Shell_log "gitlab-rake execute success."
    else
        Shell_log "gitlab-rake execute error"
    fi

    find ${Gitbak_Path} -name "*gitlab_backup.tar" -mtime +7 -delete

}

Main;
``````

#添加定时任务，crontab -e



`30 00 * * * sh /var/opt/gitlab/myshell/gitlab_back.sh`

#### 一，宿主机上实行定时任务

**注意：docker命令不需要加**`-it`**，因为加**`-it`**就要开启了一个终端，而计划任务是无法进入任何终端。**

```
#添加定时任务
crontab -e 
30 00 * * * docker exec gitlab /bin/bash /var/opt/gitlab/myshell/gitlab_bak.sh


# 手动执行情况
[root@disp-parent backups]# docker exec -it gitlab /bin/bash /var/opt/gitlab/myshell/gitlab_bak.sh
Dumping database ... 
Dumping PostgreSQL database gitlabhq_production ... [DONE]
done
Dumping repositories ...
 * java/VRBP_Admin ... [DONE]
 * woasis-java/VRBP_Web ... [DONE]
 * woasis-java/vrbp-charging ... [DONE]
 * woasis-java/vrbp-vrbs ... [DONE]
 * woasis-java/woasis_vrbp_mpush ... [DONE]
 * woasis-java/woasis-vrbp-hisds ... [DONE]
 * woasis-java/woasis-vrbp-synchro ... [DONE]
 * woasis-java/vrbp-schedule-tasktracker ... [DONE]
 * woasis-java/VRBP_Common ... [DONE]
 * woasis-java/uap-open ... [DONE]
 * woasis-java/VRBP-Api ... [DONE]
 * woasis-java/VRBP_Admin ... [DONE]
 * teamwork/woasis-project-notes ... [DONE]
 * ops/nodejs-test ... [DONE]
 * woasis-android/WoasisPandAuto ... [DONE]
 * woasis-ios/WoasisPandAuto ... [DONE]
 * java/xp-vehicle-network ... [DONE]
 * ops/pd_shell ... [DONE]
 * ops/keys ... [SKIPPED]
 * java/xp-member ... [DONE]
 * java/xp-cms ... [DONE]
 * java/verification-code-service ... [DONE]
 * wangwei/baidu-summit-api ... [DONE]
 * java/xp-h5-api ... [DONE]
 * java/xp-coupon-service ... [DONE]
 * java/xp-coupon-service.wiki ...  [SKIPPED]
 * java/xp-payment ... [DONE]
 * java/xp-open-api ... [DONE]
 * java/xp-illegal ... [DONE]
 * huangyu/duola-parent ... [DONE]
 * huangyu/duola-parent.wiki ...  [DONE]
 * huangyu/schedule-parent ... [DONE]
 * java/xp-extend ... [DONE]
 * ops/admin ... [DONE]
 * java/xp-auto-drive ... [DONE]
 * java/xp-activity-h5 ... [DONE]
 * java/lz-simulator ... [DONE]
 * architecture/duola-transaction ... [DONE]
 * architecture/duola-transaction.wiki ...  [DONE]
 * iot/panda-mqtt-broker ... [DONE]
 * iot/moquette ... [DONE]
 * iot/mqttwk ... [DONE]
 * iot/mqtt-client ... [DONE]
 * dispatch-java/xp-dispatcher-parent ... [DONE]
 * dispatch-java/xp-dispatcher-parent.wiki ...  [SKIPPED]
 * dispatch-web/xp-dispatcher-admin-h5 ... [DONE]
 * rentcar-web/xp-common-h5 ... [DONE]
 * rentcar-ios/rentcar-ios ... [DONE]
 * zhangmin/rentcar-android ... [SKIPPED]
 * dispatch-ios/Work ... [DONE]
 * dispatch-java/xp-dispatcher-api ... [DONE]
 * dispatch-android/Work ... [DONE]
 * rentcar-ios/PandAuto-iOS ... [DONE]
 * rentcar-web/xp-wap2.0-h5 ... [DONE]
 * rentcar-web/xp-wechat-MiniProgram ... [DONE]
 * dispatch-ios/Dispatcher ... [DONE]
 * rentcar-web/xp-alipay-MiniProgram ... [DONE]
 * rentcar-web/xp-activity ... [DONE]
 * dispatch-ios/Dispatcher-Swift ... [DONE]
 * rentcar-web/xp-qiyu-customer ... [DONE]
 * rentcar-android/PandAuto-Android ... [DONE]
 * rentcar-web/xp-demandManagement-h5 ... [DONE]
 * dispatch-android/Dispatcher ... [DONE]
 * dispatch-java/xp-dispatcher-admin ... [DONE]
 * dispatch-java/xp-dispatcher-oracle-rmi ... [DONE]
 * dispatch-web/xp-dispatch-h5 ... [DONE]
 * rentcar-web/Pd-web ... [DONE]
 * java/cat-xpand ... [DONE]
 * java/xp-miniProgram-api ... [DONE]
 * dispatch-web/appH5 ... [SKIPPED]
 * rentcar-web/xp-app-h5 ... [DONE]
 * huangyu/schedule-activiti-web ... [DONE]
 * java/xp-identifyCarDamage ... [DONE]
 * iot/nimble-form ... [DONE]
 * iot/wechat-mini-control ... [DONE]
 * iot/pd-iov-platform ... [DONE]
 * iot/panda-login_3_19 ... [DONE]
 * iot/panda-platform_3_19 ... [DONE]
 * iot/panda-login_3_20 ... [DONE]
 * iot/panda-platform_3_20 ... [DONE]
 * iot/panda-platform_4_11 ... [DONE]
 * iot/panda-platform ... [DONE]
 * iot/panda-login ... [DONE]
 * woasis-java/xpand-rmi-starter ... [DONE]
 * liubo/xp-dispatcher-workflow-bpmn ... [DONE]
 * dispatch-java/xp-common-utils ... [DONE]
 * java/xp-job ... [DONE]
 * java/xp-sms-api ... [DONE]
 * root/dev-ops-docker-backup ... [DONE]
done
Dumping uploads ... 
done
Dumping builds ... 
done
Dumping artifacts ... 
done
Dumping pages ... 
done
Dumping lfs objects ... 
done
Dumping container registry images ... 
[DISABLED]
Creating backup archive: 1589080880_2020_05_10_gitlab_backup.tar ... done
Uploading backup archive to remote storage  ... skipped
Deleting tmp directories ... done
done
done
done
done
done
done
done
Deleting old backups ... skipping

# 查看备份文件
[root@disp-parent backups]# ls /data/gitlab/opt/backups/
1588946908_2020_05_08_gitlab_backup.tar  1589080880_2020_05_10_gitlab_backup.tar
```

##### 三 docker内cron（不推荐，这里只给思路，原因：1需要更高权限--private，给与docker容器更多的权限包括对系统时间的修改等，不安全，2，复杂，需要修改系统内文件，导致镜像系统冗余）

#添加阿里云源

```
#查看系统版本
root@0f4c4f917148:/etc/default# cat /etc/issue
Ubuntu 16.04.6 LTS \n \l
#备份源source文件
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
#修改source为阿里云源
root@0f4c4f917148:/etc/default# cat /etc/apt/sources.list
deb-src http://archive.ubuntu.com/ubuntu xenial main restricted #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-updates main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates universe
deb http://mirrors.aliyun.com/ubuntu/ xenial multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-updates multiverse
deb http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-backports main restricted universe multiverse #Added by software-properties
deb http://archive.canonical.com/ubuntu xenial partner
deb-src http://archive.canonical.com/ubuntu xenial partner
deb http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted
deb-src http://mirrors.aliyun.com/ubuntu/ xenial-security main restricted multiverse universe #Added by software-properties
deb http://mirrors.aliyun.com/ubuntu/ xenial-security universe
deb http://mirrors.aliyun.com/ubuntu/ xenial-security multiverse
# 更新source文件
apt-get update
apt-get upgrade
#安装cron
apt-get install cron
#第一次使用crontab会选择编辑器，选择2,使用vim，1是nano编辑器，我不太会用。
#添加 定时脚本信息
*/20 * * * * /sbin/ntpdate pool.ntp.org > /dev/null 2>&1
30 00 * * * sh /var/opt/gitlab/myshell/gitlab_back.sh
```







