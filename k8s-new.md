k8s-二进制部署
#服务器环境
    yum install -y conntrack ntpdate ntp ipvsadm ipset jq iptables curl sysstat libseccomp wget vim net-tools git
##关闭防火墙并设置为iptables并设置为空规则
    systemctl stop firewalld && systemctl disable firewalld
    yum -y install iptables-services && systemctl start iptables && systemctl enable iptables && iptables -F && service iptables saveG
##关闭selinux和swap
    swapoff -a && sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
    setenforce 0 && sed -i 's/^SELINUX=.*/SELINUX=disabled/' /etc/selinux/config
##关闭不必要系统服务postfix
    systemctl stop postfix && systemctl disable postfix
##配置系统时区并安装ntp同步时间
    # 设置系统时区为中国/上海
    timedatectl set-timezone Asia/Shanghai
    # 将当前的 UTC 时间写入硬件时钟
    timedatectl set-local-rtc 0
    # 重启依赖于系统时间的服务
    systemctl restart rsyslog
    systemctl restart crond
    #同步ntp(没安装需要安装)
    ntpdate -u cn.pool.ntp.org
    crontab -e
    59 23 * * * ntpdate -u cn.pool.ntp.org
    
##设置rsyslogd和systemd journald
    mkdir /var/log/journal 
    # 持久化保存日志的目录
    mkdir /etc/systemd/journald.conf.d
    vim /etc/systemd/journald.conf.d/99-prophet.conf
    [Journal]
    # 持久化保存到磁盘
    Storage=persistent
    # 压缩历史日志
    Compress=yes
    SyncIntervalSec=5m
    RateLimitInterval=30s
    RateLimitBurst=1000
    # 最大占用空间 
    SystemMaxUse=10G
    # 单日志文件最大 
    SystemMaxFileSize=200M
    # 日志保存时间 2 周
    MaxRetentionSec=2week
    # 不将日志转发到 syslog
    ForwardToSyslog=no

    systemctl restart systemd-journald


#######配置免密master
########安装docker
    https://docs.docker.com/install/linux/docker-ce/centos/
##升级内核
    安装elrepo源
    [root@host-10-20-89-5 default]# rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
    [root@host-10-20-89-5 default]# yum install https://www.elrepo.org/elrepo-release-7.0-3.el7.elrepo.noarch.rpm

    原文链接：https://blog.csdn.net/warylee/article/details/93979154
    # 安装完成后检查 /boot/grub2/grub.cfg 中对应内核 menuentry 中是否包含 initrd16 配置，如果没有，再安装一次
    yum --disablerepo="*" --enablerepo="elrepo-kernel" list available
    yum --enablerepo=elrepo-kernel install kernel-lt 
    #查看新内核
    awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
    [root@localhost ~]# awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
    0 : CentOS Linux (4.4.215-1.el7.elrepo.x86_64) 7 (Core)
    1 : CentOS Linux (3.10.0-1062.12.1.el7.x86_64) 7 (Core)
    2 : CentOS Linux (3.10.0-1062.el7.x86_64) 7 (Core)
    3 : CentOS Linux (0-rescue-b6ea4b8026264e5ab3377ffa645db044) 7 (Core)
    #将新内核设置为default
     vi /etc/default/grub
    修改GRUB_DEFAULT=0
    
    #重新创建kernel配置
    grub2-mkconfig -o /boot/grub2/grub.cfg
    #重启
    reboot
