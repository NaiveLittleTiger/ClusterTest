# NFS 服务器、NIS 服务器 、以及 Torque作业管理系统安装
## 写在前面
此次内容是在三台台弃用的服务器进行测试学习，一台作为master节点，其中两台slave节点。

在开始操作之前，需要将**三台机器进行免密登录设置，以及关闭防火墙和SELINUX**。

## 一、NFS 安装
## 1.1 NFS 服务端 server

### 1.1.1、安装软件安装包
```
yum -y install nfs-utils
```
### 1.1.2 、启动服务
```
systemctl start rpcbind
systemctl start nfs
```
### 1.1.3、设置开机自启动
```
systemctl enable rpcbind
systemctl enable rpcbind
```
### 1.1.4、设置配置文件/etc/exports，设置共享目录为/public（需要提前创建）以及主机网段
```
# 写上自己的主机ip/子网掩码，以及设置共享文件
/public 192.168.2.0/24(ro,sync,no_root_squash)
```
### 1.1.5、使用exportfs命令进行挂载 参数r重新挂载/etc/exports里面的设置，v将共享的目录显示
```
exportfs -rv
```

## 1.2 NFS 客户端 client

### 1.2.1、安装软件安装包
```
yum -y install nfs-utils
```
### 1.2.2、启动服务
```
systemctl start rpcbind
systemctl start nfs
```
### 1.2.3、设置开机自启动
```
systemctl enable nfs
systemctl enable rpcbind
```
### 1.2.4、挂载admin的/public共享目录到该主机的/public 使用mountfs命令
```
mkdir /public
mountfs -t admin:/public /public
#有时mountfs命令没有，可以使用mount nfs代替
mount -t nfs admin:/public /public
#挂载成功后使用df检查
df
```

## 二、NIS 服务器安装
## 2.1 服务端安装
### 2.1.1、安装nis相关软件包
```
yum -y install ypserv
yum -y install ypbind
yum -y install yp-tools
```
### 2.2.2、设置nis域名
```
nisdomainname #显示当前的nis域
nisdomainname tigeryao #设置nis域名
```

vi /etc/rc.d/rc.local   #设置开机自启动（rc.local一定要可执行） 加入一行 vi /etc/rc.local也一样
#加入下面内容
```
/bin/nisdomainname tigeryao
#设置在启动nis时就设置好nis域
vi /etc/sysconfig/network
NISDOMAIN=tigeryao
```
### 2.2.3、主要配置文件/etc/ypserv.conf（有时直接默认）
```
vi /etc/ypserv.conf
```
### 2.2.4、设置主机名称对应
```
vi /etc/hosts
设置域名 和主机名对应
```
### 2.2.5、启动所有相关服务(并设置成开机自启动)
```
systemctl start rpcbind, ypserv, yppasswdd
systemctl enable rpcbind, ypserv, yppasswdd
#启动完毕后，使用rpcinfo -p
```
### 2.2.6、建立数据库
```
/usr/lib64/yp/ypinit -m 
systemctl restart ypserv
#当账号信息有更新时 cd /var/yp，make一下或者将上述命令再执行一次
```

## 2.2NIS客户端的安装
### 2.2.1、安装nis相关安装包
```
yum -y install ypbind
yum -y install yp-tools
yum -y install portreserve
```
### 2.2.2、加入nis域（开机自启动）
```
nisdomainname #显示当前的nis域
nisdomainname tigeryao #设置nis域名
```
vi /etc/rc.d/rc.local   #设置开机自启动（rc.local一定要可执行） 加入一行 vi /etc/rc.local也一样
```
#加入下面内容
/bin/nisdomainname tigeryao
#设置在启动nis时就设置好nis域
vi /etc/sysconfig/network
NISDOMAIN=tigeryao
```
### 2.2.3、设置主机名称和ip相对应
```
vi /etc/hosts
```
### 2.2.4、启动ypbind连接至nis server（authconfig）图形化界面设置（也可以手动设置）
```
authconfig 
systemctl restart rpcbind
systemctl restart ypbind
#开机自启动
systemctl enable rpcbind
systemctl enable ypbind
```
### 2.2.5、nis 客户端的检验
#在服务端admin usradd 新用户，测试能否在
yptest，ypcat

**总结，本次出错原因，在服务端运行了ypbind，只需要在客户端运行ypbind即可**

## 三、Torque作业管理系统的安装
## 3.1 Master节点的安装
### 3.1.1、安装和配置torque
```
cd /opt
mkdir torque
cd torque
wget http://wpfilebase.s3.amazonaws.com/torque/torque-6.1.2.tar.gz
tar -zxvf torque
```
### 3.1.2、安装torque依赖包
```
yum install libxml2-devel openssl-devel gcc gcc-c++ boost-devel libtool-y
```
### 3.1.3、设置安装信息
```
# master 需要改为你自己的pbs server
./configure --prefix=/usr/local/torque --with-scp --with-default-server=master
```

### 3.1.4、编译、安装，打包
```
make && make install && make packages
```
#将contrib/init.d/目录下的pbs_server、pbs_sched、pbs_mom、trqauthd添加到系统初始化简脚本/etc/init.d/中，并设置为开机启动。
```
cp contrib/init.d/{pbs_{server,sched,mom},trqauthd} /etc/init.d/
for i in pbs_server pbs_sched pbs_mom trqauthd; do chkconfig --add $i; chkconfig $i on; done
```
### 3.1.5、设置环境变量
```
TORQUE=/usr/local/torque  
echo "TORQUE=$TORQUE" >>/etc/profile
echo "export PATH=\$PATH:$TORQUE/bin:$TORQUE/sbin" >>/etc/profile
source /etc/profile
```
### 3.1.6、设置TORQUE的管理用户，设置为非root用户（可选设置）
```
./torque.setup user1
```

### 3.1.7、启动pbs_server、pbs_sched、pbs_mom、trqauthd几个服务
```
qterm
for i in pbs_server pbs_sched pbs_mom trqauthd; do service $i restart; done
# 启动成功使用 ps 命令查看
ps -e | grep pbs
```
### 3.1.8 创建 students 队列

```
qmgr
create queue students
set queue students queue_type = Execution
set queue students Priority = 40
set queue students resources_max.cput = 96:00:00
set queue students resources_min.cput = 00:00:01
set queue students resources_default.cput = 96:00:00
set queue students enabled = True
set queue students started = True
```

## 3.2 Salves节点安装

### 3.2.1、将master节点下torque-6.1.1.1的torque-package*文件copy到salve1节点torque6中
```
scp torque-package-{mom,clients}-linux-x86_64.sh salve1:torque
```
### 3.3.2、将master节点下torque-6.1.1.1的contrib/init.d/{pbs_mom,trqauthd}文件copy到salve1节点/etc/init.d/
```
scp contrib/init.d/{pbs_mom,trqauthd} salve1:/etc/init.d/
```
### 3.3.3、安装拷贝文件
```
./torque-package-clients-linux-x86_64.sh --install  
./torque-package-mom-linux-x86_64.sh --install  
```
### 3.3.4、创建/var/spool/torque/mom_priv/config文件
```
vi /var/spool/torque/mom_priv/config
pbsserver master #写上自己的主机名
logevent 225   #写上需要记录那些日志文件的bitmap
```


### 3.3.5、将pbs_mom和trqauthd设置开机启动
```
for i in pbs_mom trqauthd; do chkconfig --add $i; chkconfig $i on; done
```
### 3.3.6、将计算节点加入到服务节点master中,切换到master节点
#编辑/var/spool/torque/server_priv/nodes文件并写入如下内容
```
salve1 np=1
```
### 3.3.7、启动salve1节点的pbs_mom
```
for i in pbs_mom trqauthd; do service $i start; done
```
### 3.3.8、查看新增计算节点salve1
查看节点状态
```
qnodes
```
### 3.3.9、安装MAUI 作业调度软件
- 官网（maui:http://www.adaptivecomputing.com/support/download-center/maui-cluster-scheduler/） 下载安装包
- 编译安装
```
tar -xzvf maui-3.2.6.tar.gz
cd maui-3.2.6
./configure
make && make install
```
- 加入路径, 添加环境变量
```
vi /etc/profile
export PATH=/usr/local/maui/bin/:/usr/local/maui/sbin/:$PATH
```
- 配置maui.cfg
- 启动
  ```
  maui
  ```
- 写入/etc/rc.local
  ```
  /usr/local/maui/sbin/maui
  ```


## 四、错误排查

- **一直遇到报错，*log_error::send_update_to_a_server,colud not contact any of the servers to send an update*
后在stackoverflow上找到是防火墙的原因**
```
 iptables-save > iptables.bak && iptables -F
 ```
指令可以将内核中当前的iptables配置导出到标准输出，>覆盖输出到文本，>>追加到文本

iptables.bak是iptables-save命令所备份的文件，iptables -F清空

- 后续使用qsub报错 *Bad UID for job execution MSG=ruserok failed validating*
解决方法
**需要设置为允许在节点提交**
```
qmgr -c 'set server submit_hosts = submission_node'
qmgr -c 'set server allow_node_submit = True'
#然后 重启pbs_server
service pbs_server start
```
