---
title: Centos 搭建 cephfs 集群
date: 2019-07-08 02:27:14
tags: [cephfs, 分布式文件系统]
categories: 运维
---

## 部署准备
1. 准备服务器，本实验准备三各节点
   ```
   192.168.0.177    ceph-admin  # 作为ceph-deploy， Monitor(mon)， Metadata(mds)
   192.168.0.179    ceph-node1  # 作为 OSD (object storage daemon)
   192.168.0.157    ceph-node2  # 作为 OSD (object storage daemon)
   ```

2. 分别设置各服务器名称
   ```
   $ hostnamectl set-hostname ceph-admin
   $ hostnamectl set-hostname ceph-node1
   $ hostnamectl set-hostname ceph-node2
   ```

3. 在各个机器上执行：追加 hosts 各个主机名称以及IP，方便后期相互通过主机名称进行连接
   ```
   $ cat << EOF >> /etc/hosts
   192.168.0.177    ceph-admin
   192.168.0.179    ceph-node1 
   192.168.0.157    ceph-node2
   EOF
   ```

4. 每个节点关闭防火墙，关闭SELinux
   ```
   $ systemctl stop firewalld
   $ systemctl disable firewalld
   $ sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config   # 永久生效
   $ setenforce 0  #临时生效
   ```

5. 每个节点安装NTP，以免因时钟漂移导致故障
   
   ```
   $ yum install ntp ntpdate ntp-doc -y
   $ systemctl restart ntpd
   $ systemctl status ntpd
   ```
   **note**: 若无法联网，则需要在某一节点上安装ntp-server，其他ntp与此 ntp-server进行时间同步

6. 每个节点创建新用户

   ```
   $ useradd -d /home/ai -m ai
   $ passwd ai 
   # 输入ai用户密码
   # 确保各 Ceph 节点上新创建的用户都有 sudo 权限
   $ echo "ai ALL = (root) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/ai
   $ sudo chmod 0440 /etc/sudoers.d/ai
   ```
   **note**: 新建用户名随意，但不要用 “ceph” 这个名字。因为用户名 “ceph” 保留给了 Ceph 守护进程

7. SSH 免密登录
   允许上步骤新建的用户ai，无密码 SSH 登录
   
   假设在ceph-admin上执行：
   ```
   # 切换 ai用户
   $ su - ai
   $ ssh-keygen   #一路回车
   # 将生成的文件 /ai/.ssh/ 拷贝到其他节点
   $ ssh-copy-id ceph-node1  # 根据提示，输入ceph-node1的ai用户密码
   $ ssh-copy-id ceph-node2  # 根据提示，输入ceph-node2的ai用户密码

   # 测试通过 ceph-admin 进行免密登录 ceph-node1 和 ceph-node2
   $ mkdir -p test
   $ scp -r test/ ceph-node1:~/  #该步骤无需密码就可以 scp操作
   # 切换至 ceph-node1 查看存在文件夹 test
   ```
8. 配置YUM源
   
   每个节点更改ceph所需 yum源 
   ```
   # 切换到root用户
   $ su - root  #若不切换，则在下面命令前加 sudo 
   $ yum install -y wget vim
   $ yum clean all
   $ mkdir -p /etc/yum.repos.d.bak
   $ mv /etc/yum.repos.d/*  /etc/yum.repos.d.bak/
   # 下载阿里云的base源和epel源
   $ wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
   $ wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
   $ vim /etc/yum.repos.d/ceph.repo  #将下面内容粘贴进去
   [ceph]
   name=ceph
   baseurl=https://mirrors.aliyun.com/ceph/rpm-jewel/el7/x86_64/
   gpgcheck=0
   priority =1
   [ceph-noarch]
   name=cephnoarch
   baseurl=https://mirrors.aliyun.com/ceph/rpm-jewel/el7/noarch/
   gpgcheck=0
   priority =1
   [ceph-source]
   name=Ceph source packages
   baseurl=https://mirrors.aliyun.com/ceph/rpm-jewel/el7/SRPMS
   gpgcheck=0
   priority=1
   ```


## 部署ceph集群

1. 部署ceph-deploy
   ```
   # 切换ai用户
   $ su - ai
   # 在用户目录创建ceph，用于存储 ceph-deploy 产生的配置
   $ mkdir -p cluster && cd cluster
   # 安装 ceph-deploy
   $ sudo yum install ceph-deploy -y
   ```
2. 创建集群
   ```
   # 创建集群（后面填写monit节点的主机名，这里monit节点和管理节点是同一台机器，即ceph-admin）
   $ sudo ceph-deploy new ceph-admin   # 该参数是monitor的节点名称
   $ [ai@ceph-admin cluster]$ ll
     total 12
     -rw-rw-r--. 1 ai ai  201 Nov 22 09:46 ceph.conf
     -rw-rw-r--. 1 ai ai 3087 Nov 22 09:46 ceph-deploy-ceph.log
     -rw-------. 1 ai ai   73 Nov 22 09:46 ceph.mon.keyring
   ```
   修改ceph.conf文件（注意：mon_host必须和public network 网络是同网段内！）
   ```
   $[ai@ceph-admin cluster]$ sudo vim ceph.conf     #添加下面两行配置内容 
   mon_initial_members = ceph-admin
   mon_host = 192.168.0.177
   auth_cluster_required = cephx
   auth_service_required = cephx
   auth_client_required = cephx
   public network = 192.168.0.177/24     # mon_host必须和public network 网络是同网段内
   osd pool default size = 2
   ```
   安装ceph
   ```
   [ai@ceph-admin cluster]$ sudo ceph-deploy install ceph-admin ceph-node1 ceph-node2
   ```
   **NOTE**: 这里会在每个节点生成 ceph.repo文件，里面配置了官方的 `http://download.ceph.com/rpm-jewel/el7/` 源，但是对于离线环境或者网络不佳的情况，执行上述install操作，会报超时错误，导致安装失败；解决办法：请手动删除自动生成的ceph.repo 。并配置上述的aliyun的源；手动在各个节点执行： `sudo yum install -y ceph ceph-radosgw`  后，再进行 `ceph-deploy install ceph-admin ceph-node1 ceph-node2` 的安装。


   
   初始化monit监控节点，并收集所有密钥
   ```
   [ai@ceph-admin cluster]$ sudo ceph-deploy --overwrite-conf mon create-initial 
   [ai@ceph-admin cluster]$ sudo chmod 644 *
   [ai@ceph-admin cluster]$ sudo ceph-deploy  gatherkeys ceph-admin
   [ceph_deploy.conf][DEBUG ] found configuration file at: /home/ai/.cephdeploy.conf
   [ceph_deploy.cli][INFO  ] Invoked (1.5.39): /bin/ceph-deploy    gatherkeys ceph-admin
          ... ...
          ... ...
   [ceph_deploy.gatherkeys][INFO  ] keyring    'ceph.bootstrap-rgw.keyring' already exists
   [ceph_deploy.gatherkeys][INFO  ] Destroy temp directory /tmp/   tmpG_r7Y6
   

   ```
   **NOTE**: 在初始化monit监控时，可能会报如下错误，按照提示覆盖已存在配置即可；
   ```
   [ceph-admin][DEBUG ] write cluster configuration to /etc/ceph/{cluster}.conf
   [ceph_deploy.mon][ERROR ] RuntimeError: config file /etc/ceph/   ceph.conf exists with different content; use --overwrite-conf    to overwrite
   [ceph_deploy][ERROR ] GenericError: Failed to create 1 monitors
   ```
 

   在各个节点上创建 OSD 使用的目录
   ```
   $ sudo mkdir -p /data/ceph-data
   # 修改该目录为 ceph 用户所有
   $ sudo chown -R ceph:ceph /data/ceph-data
   
   ```


   
   准备OSD (prepare)
   ```
   $ sudo chown -R ceph:ceph /etc/ceph/
   $ sudo ceph-deploy --overwrite-conf  osd prepare ceph-node1:/data/ceph-data ceph-node2:/data/ceph-data
   ...
   ...
   [ceph-node2][DEBUG ] find the location of an executable
   [ceph-node2][INFO  ] Running command: sudo /bin/ceph --cluster=ceph osd    stat --format=json
   [ceph_deploy.osd][DEBUG ] Host ceph-node2 is now ready for osd use.
   
   ```

   激活 OSD
   ```
   $ sudo ceph-deploy osd activate ceph-node1:/data/ceph-data ceph-node2:/data/ceph-data
   ```
   
   用ceph-deploy把配置文件和admin密钥拷贝到管理节点和Ceph节点，这样你每次执行Ceph命令行时就无需指定monit节点地址和ceph.client.admin.keyring了
   ```
   $ sudo ceph-deploy admin ceph-admin ceph-node1 ceph-node2
   ```
   进入到**每个节点**的/etc/ceph下，修改 ceph.client.admin.keyring 的读权限
   ```
   $ sudo chmod 644 /etc/ceph/*
   ```
   查看集群健康状况
   ```
   $ sudo ceph healthy
   HEALTH_OK
   ```

   查看ceph集群状态 status
   ```
   $ sudo ceph -s
   cluster c38b9cf8-582d-43d9-900a-9f25525fd7d7
     health HEALTH_OK
     monmap e1: 1 mons at {ceph-admin=192.168.0.177:6789/0}
            election epoch 3, quorum 0 ceph-admin
     osdmap e10: 2 osds: 2 up, 2 in
            flags sortbitwise,require_jewel_osds
      pgmap v40: 64 pgs, 1 pools, 0 bytes data, 0 objects
            13147 MB used, 987 GB / 999 GB avail
                  64 active+clean

   ```
   查看ceph osd运行状态
   ```
   $ sudo ceph osd stat
   osdmap e10: 2 osds: 2 up, 2 in
        flags sortbitwise,require_jewel_osds
   ```

   查看osd的目录树

   ```
   $ sudo ceph osd tree
   ID WEIGHT  TYPE NAME           UP/DOWN REWEIGHT PRIMARY-AFFINITY
   -1 0.97659 root default
   -2 0.48830     host ceph-node1
    0 0.48830         osd.0            up  1.00000          1.00000
   -3 0.48830     host ceph-node2
    1 0.48830         osd.1            up  1.00000          1.00000
   ```
   分别查看mon服务、OSD服务的status 
   ```
   [root@ceph-admin ceph]# systemctl status ceph-mon@ceph-admin   # 查看 Mon 服务状态
   [root@ceph-node1 ceph]# systemctl status ceph-osd@0.service    # 查看 OSD 服务状态
   [root@ceph-node2 ceph]# systemctl status ceph-osd@1.service    # 查看 OSD 服务状态
   ```
3. 创建文件系统
   
   创建 MDS (Metadata System)
   
   如 #部署准备 述将管理节点作为元数据管理节点

   ```
   [ai@ceph-admin ~]$ cd cluster
   [ai@ceph-admin cluster]$ sudo ceph-deploy mds create ceph-admin
   # 查看 mds 服务状态
   [ai@ceph-admin cluster]$ sudo systemctl status ceph-mds@ceph-admin
   [ai@ceph-admin cluster]$ sudo ceph mds stat
   e2:, 1 up:standby
   ```

   创建 pool ， pool 是 ceph 存储数据时的逻辑分区,它起到 namespace 的作用
   新创建的 ceph 集群只有 rdb 一个 pool 。这时需要创建一个新的 pool

   创建第二个 pool cephfs_data,其有归置组（PG） 100个
   ```
   [ai@ceph-admin cluster]$ sudo ceph osd pool create cephfs_data 100       #后面的数字是PG(归置组)的数量
   pool 'cephfs_data' created
   ```

   创建第二个 pool cephfs_metadata,其有归置组（PG） 100个
   ```
   [ai@ceph-admin cluster]$ sudo ceph osd pool create cephfs_metadata 100     #创建pool的元数据
   pool 'cephfs_metadata' created
   ```

   查看存储池
   ```
   [ai@ceph-admin cluster]$ sudo ceph osd lspools
   0 rbd,1 cephfs_data,2 cephfs_metadata,   # 有三个pools(rbd, cephfs_data, cephfs_metadata)
   ```
   
   通过 metadata 和 data 创建一个文件系统(make new filesystem using named pools)
   ```
   # ceph fs new <fs_name> <metadata> <data> 
   [ai@ceph-admin cluster]$ sudo ceph fs new myceph cephfs_metadata cephfs_data
   new fs with metadata pool 2 and data pool 1
   ```
   
   查看 mds 状态
   ```
   [ai@ceph-admin cluster]$ sudo ceph mds stat
   e5: 1/1/1 up {0=ceph-admin=up:active}

   ```
   查看ceph集群状态
   ```
   [ai@ceph-admin cluster]$ sudo ceph  -s
    cluster c38b9cf8-582d-43d9-900a-9f25525fd7d7
     health HEALTH_OK
     monmap e1: 1 mons at {ceph-admin=192.168.0.177:6789/0}
            election epoch 3, quorum 0 ceph-admin
      fsmap e5: 1/1/1 up {0=ceph-admin=up:active}
     osdmap e15: 2 osds: 2 up, 2 in
            flags sortbitwise,require_jewel_osds
      pgmap v186: 264 pgs, 3 pools, 2068 bytes data, 20 objects
            13148 MB used, 987 GB / 999 GB avail
                 264 active+clean
   ```
 

## client端挂载ceph存储
   - 采用fuse方式
     a. 安装fuse客户端
       ```
       $ sudo yum install -y ceph-fuse
       ```

     b. 对客户端进行授权：
       1. 添加配置文件 `ceph.conf` 到 `/etc/ceph` 目录下:
       2. 添加秘钥 `ceph.client.admin.keyring` 到客户端目录 `/etc/ceph` 下
     
     c. 将ceph集群存储挂载到客户机的/mnt/ceph-client目录下
       ```
       [ai@ceph-node2 ceph-client]$ sudo ceph-fuse -m 192.168.0.177:6789 /mnt/ceph-client
       2019-11-26 08:13:44.818591 7f1c873b9f00 -1 init, newargv = 0x561513fdc78  0 newargc=11ceph-fuse[
       15316]: starting ceph client
       ceph-fuse[15316]: starting fuse
       ```
     d. 查看挂载情况
       ```
       [ai@ceph-node2 ceph-client]$ df -hT
       Filesystem     Type            Size  Used Avail Use% Mounted on
       /dev/vda1      xfs             500G  6.5G  494G   2% /
       devtmpfs       devtmpfs        3.8G     0  3.8G   0% /dev
       tmpfs          tmpfs           3.9G     0  3.9G   0% /dev/shm
       tmpfs          tmpfs           3.9G   17M  3.9G   1% /run
       tmpfs          tmpfs           3.9G     0  3.9G   0% /sys/fs/cgroup
       tmpfs          tmpfs           783M     0  783M   0% /run/user/0
       tmpfs          tmpfs           783M     0  783M   0% /run/user/1001
       ceph-fuse      fuse.ceph-fuse 1000G   13G  988G   2% /mnt/ceph-client
       ```
     e. 取消挂载
       ```
       $ umount /mnt/ceph-client
       ```

   - 采用 mount -t ceph方式
     ```
     $ mount -t ceph 192.168.0.177:6789:/ -o name=admin,secret=`ceph-authtool -p /etc/ceph/ceph.client.admin.keyring` /mnt/ceph-client
     $ df -hT
     Filesystem           Type      Size  Used Avail Use% Mounted on
     /dev/vda1            xfs       500G  6.5G  494G   2% /
     devtmpfs             devtmpfs  3.8G     0  3.8G   0% /dev
     tmpfs                tmpfs     3.9G     0  3.9G   0% /dev/shm
     tmpfs                tmpfs     3.9G   17M  3.9G   1% /run
     tmpfs                tmpfs     3.9G     0  3.9G   0% /sys/fs/cgroup
     tmpfs                tmpfs     783M     0  783M   0% /run/user/0
     tmpfs                tmpfs     783M     0  783M   0% /run/user/1001
     192.168.0.177:6789:/ ceph     1000G   13G  988G   2% /mnt/ceph-client
     ```
     或者将 /etc/ceph/ceph.client.admin.keyring 文件的key提取出来到指定的 xxx.secret文件中
     ```
     $ mount -t ceph 192.168.0.177:6789:/ /mnt/ceph-client -o name=admin,secretfile=xxx.secret
     ```

     **NOTE**:
     - `192.168.0.177:6789:/`: 注意最后的是`/`, 而不是上述的 OSD 目录
     - `-o name=admin,secret`: 必须制定 cephx 认证信息
     - `ceph-authtool -p /etc/ceph/ceph.client.admin.keyring`  用于生成base64的证书

     > 取消挂载同 ceph-fuse 方式一致

## 遇到问题


1. activate 时，报错：Error EACCES: access denied
   ```
   $ ceph-deploy osd activate ceph-node1:/data/ceph-data ceph-node2:/data/ceph-data
   ```
   安装时遇到问题，已经将待挂在的目录设置为ceph:ceph用户所有。
   ```
   
   [ceph-node2][INFO  ] Running command: sudo /usr/sbin/ceph-disk -v activate     --mark-init systemd --mount /data/ceph-data
   [ceph-node2][WARNIN] main_activate: path = /data/ceph-data
   [ceph-node2][WARNIN] activate: Cluster uuid is   6bff2349-7b79-4ce3-a1b3-7641cb807690
   [ceph-node2][WARNIN] command: Running command: /usr/bin/ceph-osd --cluster=ceph     --show-config-value=fsid
   [ceph-node2][WARNIN] activate: Cluster name is ceph
   [ceph-node2][WARNIN] activate: OSD uuid is 54a93b4d-84f5-4777-8245-fde176d762ab
   [ceph-node2][WARNIN] allocate_osd_id: Allocating OSD id...
   [ceph-node2][WARNIN] command: Running command: /usr/bin/ceph --cluster ceph   --name   client.bootstrap-osd --keyring /var/lib/ceph/bootstrap-osd/ceph.keyring   osd create   --concise 54a93b4d-84f5-4777-8245-fde176d762ab
   [ceph-node2][WARNIN] Traceback (most recent call last):
   [ceph-node2][WARNIN]   File "/usr/sbin/ceph-disk", line 9, in <module>
   [ceph-node2][WARNIN]     load_entry_point('ceph-disk==1.0.0', 'console_scripts',     'ceph-disk')()
   [ceph-node2][WARNIN]   File "/usr/lib/python2.7/site-packages/ceph_disk/main.py",     line 5361, in run
   [ceph-node2][WARNIN]     main(sys.argv[1:])
   [ceph-node2][WARNIN]   File "/usr/lib/python2.7/site-packages/ceph_disk/main.py",     line 5312, in main
   [ceph-node2][WARNIN]     args.func(args)
   [ceph-node2][WARNIN]   File "/usr/lib/python2.7/site-packages/ceph_disk/main.py",     line 3443, in main_activate
   [ceph-node2][WARNIN]     init=args.mark_init,
   [ceph-node2][WARNIN]   File "/usr/lib/python2.7/site-packages/ceph_disk/main.py",     line 3263, in activate_dir
   [ceph-node2][WARNIN]     (osd_id, cluster) = activate(path, activate_key_template,     init)
   [ceph-node2][WARNIN]   File "/usr/lib/python2.7/site-packages/ceph_disk/main.py",     line 3355, in activate
   [ceph-node2][WARNIN]     keyring=keyring,
   [ceph-node2][WARNIN]   File "/usr/lib/python2.7/site-packages/ceph_disk/main.py",     line 1018, in allocate_osd_id
   [ceph-node2][WARNIN]     raise Error('ceph osd create failed', e, e.output)
   [ceph-node2][WARNIN] ceph_disk.main.Error: Error: ceph osd create failed: Command   '/  usr/bin/ceph' returned non-zero exit status 13: Error EACCES: access denied
   [ceph-node2][WARNIN]
   [ceph-node2][ERROR ] RuntimeError: command returned non-zero exit status: 1
   [ceph_deploy][ERROR ] RuntimeError: Failed to execute command: /usr/sbin/  ceph-disk   -v activate --mark-init systemd --mount /data/ceph-data
   
   ```
   
   解决：
   
   重新部署，10.2.x版本的，在部署过程中，使用了 `yum update -y ` 导致系统版本升级(原来版 本  centos7.4，升级后版本centos7.7)，并且未在ceph-node1 和 ceph-node2上配置 yum源 （在yum源  中指定ceph版本为jewel），在安装时安装了最新的版本（12.XX.xx）。通过清理，重 新安装jewel版  本解决。

2. 挂载时报错：mount error 2 = No such file or directory
   ```
   $ mount -t ceph 192.168.0.177:6789:/data/ceph-data -o name=admin,   secret=`ceph-authtool -p /etc/ceph/ceph.client.admin.keyring` /mnt/ceph-client
   mount error 2 = No such file or directory
   ```
   解决：
   `mount -t ceph 192.168.0.177:6789:/data/ceph-dat` ceph集群必须是：`mount -t ceph    192.168.0.177:6789:/` 这种写法，不是想当然的加上 OSD 的服务存储目录


3. 挂载时报错：mount error 22 = Invalid argument
   ```
   $ mount -t ceph 192.168.0.177:6789:/ /mnt/ceph-client      
   mount error 22 = Invalid argument
   ```
   解决：
   1. 整除
      ```
      $ cat /etc/ceph/ceph.client.admin.keyring
      [client.admin] 
      AQD5ZORZsJYbMhAAoGEw/H9SGMpEy1KOz0WsQQ==
      # 可知用户名：admin
      # 密钥 ：AQD5ZORZsJYbMhAAoGEw/H9SGMpEy1KOz0WsQQ==
      # 之后在当前目录创建一个文件admin.secret保存该秘钥：
      $ vi admin.secret
      # 把该秘钥粘贴过来，：wq保存
      $ mount -t ceph 192.168.0.177:6789:/ -o name=admin,secretfile=admin.secret / mnt/  ceph-client
      ```
   2. 使用ceph-authtool工具
      ```
      $ mount -t ceph 192.168.0.177:6789:/ -o name=admin,secret=`ceph-authtool -p / etc/  ceph/ceph.client.admin.keyring` /mnt/ceph-client
      ```
   

     
     
