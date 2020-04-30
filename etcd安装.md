#安装etcd集群服务
    hostname          :     ip
    k8s-master11            10.4.7.11
    k8s-master12            10.4.7.12
    k8s-node22              10.4.7.22

##准备证书
    #在证书服务器上执行
    #前提是之前创建了根证书/opt/cert.
    ##创建 CA 配置文件（ca-config.json）
    [root@k8s-other200 certs]# cat ca-config.json
    {
        "signing": {
            "default": {
                "expiry": "175200h"
            },
            "profiles": {
                "server": {
                    "usages": [
                        "signing",
                        "key encipherment",
                        "server auth",
                        "client auth"
                    ],
                    "expiry": "175200h"
                },
                "client": {
                    "usages": [
                        "signing",
                        "key encipherment",
                        "client auth"                                 
                    ],
                    "expiry": "175200h"
                },
                "peer": {
                    "usages": [
                        "signing",
                        "key encipherment",
                        "server auth",
                        "client auth"
                    ],
                    "expiry": "175200h"
                }
            }
        }
    }

    #创建 CA 证书签名请求（etcd-peer-csr.json） 
    [root@k8s-other200 certs]# cat etcd-peer-csr.json 
    {
        "CN": "k8s-etcd",
        "hosts": [
            "10.4.7.11",
            "10.4.7.12",
            "10.4.7.21",
            "10.4.7.22"
        ],
        "key": {
            "algo": "rsa",
            "size": 2048
        },
        "names": [
            {
                "C": "CN",
                "ST": "SiChuan",
                "L": "ChengDu",
                "O": "kl",
                "OU": "ops"           
            }
        ]
    }
    #生成 CA 证书和私钥
    cfssl gencert -ca=ca.pem -ca-key=ca-key.pem -config=ca-config.json -profile=peer etcd-peer-csr.json |cfssl-json -bare etcd-peer
    #生成以下文件(etcd-peer.pem, etcd-peer-key.pem,etcd-peer.csr)
    [root@k8s-other200 certs]# ll -h /opt/certs
    total 36K
    -rw-r--r--  1 root root  904 Mar 10 17:24 ca-config.json
    -rw-r--r--. 1 root root 1001 Mar  5 18:03 ca.csr
    -rw-------. 1 root root 1.7K Mar  5 18:03 ca-key.pem
    -rw-r--r--. 1 root root 1.4K Mar  5 18:03 ca.pem
    -rw-r--r--  1 root root 1.1K Mar 10 17:25 etcd-peer.csr
    -rw-r--r--  1 root root  374 Mar 10 13:45 etcd-peer-csr.json
    -rw-------  1 root root 1.7K Mar 10 17:25 etcd-peer-key.pem
    -rw-r--r--  1 root root 1.5K Mar 10 17:25 etcd-peer.pem
    -rw-r--r--. 1 root root  263 Mar  5 18:03 k8s-ca.json
    #暂时结束，安装完etcd服务后
    #创建了目录后，再复制(etcd-peer.pem, etcd-peer-key.pem,ca.pem)到相应目录下,我这里是/opt/etcd/certs
    scp /opt/certs/etcd-peer.pem root@host:/opt/etcd/certs/
    scp /opt/certs/etcd-peer-key.pem root@host:/opt/etcd/certs/
    scp /opt/certs/ca.pem root@host:/opt/etcd/certs/


##安装etcd服务
####master主机上执行
    ansible etcd -m shell -a "useradd -s /sbin/nologin -M etcd"
###下载etcdV3.4.4
    wget -P /opt/ https://github.com/etcd-io/etcd/releases/download/v3.4.4/etcd-v3.4.4-linux-amd64.tar.gz
    #解压压缩包
    tar -zxf  etcd-v3.4.4-linux-amd64.tar.gz -C /opt/etcd-v3.4.4
    #添加软连接
    ln -s /opt/etcd-v3.4.4 /opt/etcd
    #创建etcd目录
    ansible etcd -m shell -a "mkdir -p /data/etcd/etcd-server/ /data/etcd/certs"
    #复制完证书再修改权限
    ansible etcd -m shell -a "chmod 600 /opt/etcd/certs/*.pem"
    ansible etcd -m shell -a "chown -R etcd:etcd /opt/etcd-v3.4.4/"
    ansible etcd -m shell -a "mkdir -p /data/logs/etcd-server && chown -R etcd:etcd /data/logs/etcd-server/"
    #添加启动脚本
    [root@k8s-master11 etcd]# cat etcd-start.sh 
    #!/bin/bash
    HOST_NAME=`hostname`
    IP_ADDR=`ifconfig -a|grep inet|grep -v 127.0.0.1|grep -v inet6|awk '{print $2}'|tr -d "addr:"`
    
    ./etcd  --name etcd-server-${HOST_NAME} \
            --data-dir /data/etcd/etcd-server \
            --quota-backend-bytes 8000000000 \
            --cert-file ./certs/etcd-peer.pem \
            --key-file ./certs/etcd-peer-key.pem \
            --peer-cert-file ./certs/etcd-peer.pem \
            --peer-key-file ./certs/etcd-peer-key.pem \
            --trusted-ca-file ./certs/ca.pem \
            --peer-trusted-ca-file ./certs/ca.pem \
            --initial-advertise-peer-urls https://${IP_ADDR}:2380 \
            --listen-peer-urls https://${IP_ADDR}:2380 \
            --listen-client-urls https://${IP_ADDR}:2379 \
            --advertise-client-urls https://${IP_ADDR}:2379,http://127.0.0.1:2379 \
            --initial-cluster etcd-server-k8s-master12=https://10.4.7.12:2380,etcd-server-k8s-master11=https://10.4.7.11:2380,etcd-server-k8s-node22=https://10.4.7.22:2380 \
            --client-cert-auth \
            --peer-client-cert-auth \
            --log-outputs stdout    


    PS:3.4.4 有几个参数更改，比如ca-file弃用，修改为trusted-ca-file,peer-ca-file改为peer-trusted-ca-file，这个启动脚本适合3.4.4版本的etcd，其他版本不做参考。

###安装supervisor
    #安装epel
    ansible etcd -m shell -a "rpm -ivh https://mirrors.aliyun.com/epel/epel-release-latest-7.noarch.rpm"
    ansible etcd -m yum -a "name=supervisor"
    #启动supervisor
    ansible etcd -m service -a "name=supervisord state=started enabled=yes
    
    #创建supervisor etcd启动配置文件
    vim /etc/supervisord.d/etcd-server.ini
    
    [program:etcd-server]
    command=/opt/etcd/etcd-start.sh              ; the program (relative uses PATH, can take args)
    numprocs=1                    ; number of processes copies to start (def 1)
    directory=/opt/etcd                ; directory to cwd to before exec (def no cwd)
    autostart=true                ; start at supervisord start (default: true)
    autorestart=true              ; retstart at unexpected quit (default: true)
    startsecs=30                  ; number of secs prog must stay running (def. 1)
    startretries=3                ; max # of serial start failures (default 3)
    exitcodes=0,2                 ; 'expected' exit codes for process (default 0,2)
    stopsignal=QUIT               ; signal used to kill process (default TERM)
    stopwaitsecs=10               ; max num secs to wait b4 SIGKILL (default 10)
    user=etcd                   ; setuid to this UNIX account to run the program
    redirect_stderr=true          ; redirect proc stderr to stdout (default false)
    stdout_logfile=/data/logs/etcd-server/etcd_stdout.log        ; stdout log path, NONE for none; default AUTO
    stdout_logfile_maxbytes=64MB   ; max # logfile bytes b4 rotation (default 50MB)
    stdout_logfile_backups=4     ; # of stdout logfile backups (default 10)
    stdout_capture_maxbytes=1MB   ; number of bytes in 'capturemode' (default 0)
    stdout_events_enabled=false   ; emit events on stdout writes (default false)

    #发送配置文件到各个节点
    ansible etcd -m copy -a "src=/etc/supervisord.d/etcd-server.ini dest=/etc/supervisord.d/"
    #重新加载配置
    ansible etcd -m shell -a "supervisorctl reload"
    #查看状态
    ansible etcd -m shell -a "supervisorctl status"
    
    k8s-master11 | CHANGED | rc=0 >>
    etcd-server                      RUNNING   pid 49138, uptime 4:05:29
    
    k8s-master12 | CHANGED | rc=0 >>
    etcd-server                      RUNNING   pid 40848, uptime 4:05:29
    
    k8s-node22 | CHANGED | rc=0 >>
    etcd-server                      RUNNING   pid 40957, uptime 4:05:29





