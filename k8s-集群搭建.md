#k8s-集群搭建

    hostname          ip
    master11     :    10.4.7.11
    master12     :    10.4.7.12
    node21       :    10.4.7.21
    node22       :    10.4.7.22
    other200     :    10.4.7.200


#master11配置

#修改hostname
    # 修改 hostname
    hostnamectl set-hostname your-new-host-name
    # 查看修改结果
    hostnamectl status
    #  设置 hostname 解析,依次添加剩下的节点，这里是为了安装ansible为后面方便些
    echo "127.0.0.1   $(hostname)" >> /etc/hosts
    echo "10.4.7.200  other200 "   >> /etc/hosts
    .  .  .
#生成ssh-key并发送至其他节点
    #生成密钥对
    ssh-keygen -t rsa

    #复制ssh公钥到远程主机，这样ssh到远程主机的时候不需要输入密码，第一次需要输入密码和yes，如果host过多，可以使用批量导入参考
    # 发送密钥到所有管理机器ssh-copy-id
    eg_host:192.168.1.1~10
    ps:{1..10}={1,2,3,4,5,6,7,8,9,10}
    for i in {1..10}; do ssh-copy-id -i 192.168.1.$i ; done
    或者这个https://www.cnblogs.com/lizhaojun-ops/p/11059999.html
    ssh-copy-id root@10.4.7.11
    ssh-copy-id root@10.4.7.12
    ssh-copy-id root@10.4.7.21
    ssh-copy-id root@10.4.7.22
    ssh-copy-id root@10.4.7.200

    ssh的时候不会提示是否保存密钥，这里设置为host为后面配置DNS方便，依次导入
    ssh-keyscan master11 >> ~/.ssh/known_hosts
    ssh-keyscan master12 >> ~/.ssh/known_hosts
    . . .

    验证ssh是否可以连接，是否不出现提示。
    ssh root@10.4.7.200

#安装ansible
    #配置epel源
    [root@ansible ~]# wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
    [root@ansible ~]# yum clean all
    [root@ansible ~]# yum makecache

    #安装ansible
    yum -y install ansible
    #查看版本
    ansible --version

    #修改ansible配置文件，/etc/ansible/hosts,先配置ip，后面dns服务搭建好后再配置为域名
    vim /etc/ansible/hosts
    #新建一个组，因为配置密钥登陆就没有配置密码，密码可以看ansible安装
    [k8s] 
    10.4.7.11
    10.4.7.12
    10.4.7.21
    10.4.7.22
    10.4.7.200

    #测试ansible
    ansible k8s -m ping
#配置域名解析
#bind
###安装bind
    yum install bind -y

###配置bind主配置文件
    vim /etc/named.conf
    # DNS服务监听系统的 53 端口，bind配置文件对空格及分号敏感
    # 修改监听地址为本机（示例为10.4.7.11）局域网地址；
    	listen-on port 53 { 127.0.0.1; };
        #修改为
        listen-on port 53 { 10.4.7.11; };
        
    # 修改允许的使用本DNS服务器的host
    	allow-query     { localhost; }; 支持 localhost, any ,acl ...
        #修改为
        allow-query     { any; };
        
    # 添加 针对外网的上级DNS服务器地址
        forwarders      { 10.4.7.254; };
    
    # 确认 DNS查询算法类型是否为recursion，递归算法，
        recursion   yes;
    
    # 测试环境关闭dnssec：拓展：https://blog.csdn.net/huangzx3/article/details/86526068
    	dnssec-enable yes;
    	dnssec-validation yes;
        #修改为
    	dnssec-enable no;
    	dnssec-validation no;    
    # 修改完保存退出
        :wq


###检查配置文件
    named-checkconf
    
    #我这里测试     
    allow-query     { any;};少加一个空格并没有报错 删除分号后报错

    [root@k8s-master11 playbook]# named-checkconf
    /etc/named.conf:25: missing ';' before 'forwarders'


###添加域配置文件
    #修改/etc/named.rfc1912.zones  添加自己的域
    vim /etc/nameed.rfc1912.zones
    zone "host.com" IN {
        type master;
        file "host.com.zone"
        allow-update { 10.4.7.11; };
    };
    
    zone "kl.com" IN {
        type master;
        file "kl.com.zone"
        allow-update { 10.4.7.11; };
    };


###配置区域数据文件，就是自己配置的dns解析文件
    在/var/named下面创建host.com.zone,kl.com.zone添加如下配置
    vim /var/named/host.com.zone
    
    $TTL 600     ;10 minutes
    @	IN SOA	dns.host.com. kaokao55.qq.com. (
    					2020030501	; serial(YYMMDDhhmm)
    					10800       	; refresh(3 hours)
    					900         	; retry(15 minutes)
    					604800      	; expire(1 week)
    					96400       	; minimum(1 day)
                        )
    	                NS	 dns.host.com.
    $TTL 60; (1 minutes)
    dns                A	10.4.7.11
    k8s-master11   A    10.4.7.11 
    k8s-master12   A    10.4.7.12 
    k8s-node21     A    10.4.7.21 
    k8s-node22     A    10.4.7.22 
    k8s-other200   A    10.4.7.200

    #重启named服务
    systemctl start named
    测试dns解析，如果解析不通，检查named配置文件，之后再重启测试
    dig -t A k8s-master11.host.com @10.4.7.11 +short
###批量修改network dns配置并重启
    #修改dns为本服务器
    ansible k8s -m shell -a "sed -i 's/DNS1=10.4.7.254/DNS1=10.4.7.11/g' \
     /etc/sysconfig/network-scripts/ifcfg-ens33"
    #添加短域名解析，如果是生产环境不建议
    ansible k8s -m shell -a "echo "DOMAIN=host.com" >> /etc/sysconfig/network-scripts/ifcfg-ens33"
    #重启network
    ansible k8s -m shell -a "systemctl restart network"
    #在各台服务器上测试DNS
    #测试完成后可以将ansible的hosts文件中的ip修改为域名，也可以不改

    #在ansible情况下可以这个时候使用playbook执行前置环境的修改


###other200host上
#配置签发证书环境
    #下载组件
    wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64 -O /usr/bin/cfssl
    wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64 -O /usr/bin/cfssl-json
    wget https://pkg.cfssl.org/R1.2/cfssl-certinfo_linux-amd64 -O /usr/bin/cfssl-certinfo
    #添加执行权限
    chmod +x /usr/bin/cfssl*
    # 创建证书文件夹
    mkdir -p /opt/certs
###添加创建CA证书签名请求
    vim k8s-ca.json
    {
    "CN": "kubernetes",
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "CN",
            "L": "SiChuan",
            "O": "ChengDu",
            "ST": "Chengdu",            
            "OU": "ops"
        }    ],
    "ca": {
            "expriy": "175200h"
    }
    }
    
###生成ca证书
    cfssl gencert -initca k8s-ca.json|cfssl-json -bare ca
    #查看目录下是否有
    [root@k8s-other-200 certs]# ls
    ca.csr  ca-key.pem  ca.pem  k8s-ca.json

###docker环境
    #清理docke前置版本
    sudo yum remove docker \
                  docker-client \
                  docker-client-latest \
                  docker-common \
                  docker-latest \
                  docker-latest-logrotate \
                  docker-logrotate \
                  docker-engine
    

    #安装docker安装前置软件
    sudo yum install -y yum-utils devce-mapper-persistent-data
    #删除local源并安装repo源
    rm -fr /etc/yum.repos.d/local.repo
    curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
    wget -O /etc/yum.repos.d/docker-ce.repo https://mirrors.ustc.edu.cn/docker-ce/linux/centos/docker-ce.repo
    sed -i 's#download.docker.com#mirrors.tuna.tsinghua.edu.cn/docker-ce#g' /etc/yum.repos.d/docker-ce.repo"
      tags: insrepo
    #安装docker及依赖
    sudo yum install -y docker-ce docker-ce-cli containerd.io

    #创建docker目录
    mkdir -p /etc/docker
    mkdir -p /data/docker
    #创建docker-daemon文件
    vim /etc/daemon.json
    {
    "graph": "/data/docker",
    "storage-driver": "overlay2",
    "insecure-registries": ["registry.access.redhat.com","quay.io","pd-repo.xiaopankeji.com"],
    "registry-mirrors": ["https://q2gr04ke.mirror.aliyuncs.com"],
    "bip": "172.7.21.1/24",
    "exec-opts": ["native.cgroupdriver=systemd"],
    "live-restore": true
    }
    ：wq #保存退出

    #启动docker
    systemctl start docker

    #检查是否启动
     docker --version
    

###安装私有仓库harbor
    下载harbor-offline安装包
    安装python，python-util，gcc

    安装docker-compose

    修改harbor.yml
    #修改日志存放目录，端口（eg:8080）
    执行install.sh
    
    测试wget host:8080

    安装nginx作为反代理

    添加/etc/nginx/conf.d/harbor.conf
    #
    listen 80;
    server harbor.kl.com
    location {
        proxy_pass 127.0.0.1:8080;
    }
    

###修改master11dns配置，在kl.com.zone下面添加一条A记录
    vim /var/named/kl.com.zone
    

    测试


###部署master节点

####部署etcd集群
    #下载etcd，解压

    创建目录，拷贝证书，私钥
    master主机上生成证书拷贝纸节点机器上
    创建etcd证书文件夹并生成证书

    