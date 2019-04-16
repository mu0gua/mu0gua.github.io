# cuckoo + KVM

*搭建完以后敲的，有误留言，谢谢。*

------

#### 初始环境准备

> 系统版本：Ubuntu 18.4  Python：2.7  KVM

#### 宿主机环境安装

```bash
 #检查是否支持 egrep -c '(vmx|svm)' /proc/cpuinfo  虽然无diao用
 #安装依赖以及kvm
 sudo apt-get install qemu-kvm libvirt-bin virtinst bridge-utils cpu-checker
```

```bash
 apt-get update
 apt-get install -y python python-pip python-dev libffi-dev libssl-dev
 apt-get install -y python-virtualenv python-setuptools
 apt-get install -y libjpeg-dev zlib1g-dev swig
 # 如果使用web交互界面，需要安装mongodb，ubuntu 12.04安装mongodb有问题，不能简单安装，需要参考官网文档进行安装 （可以不安装）
 #apt-get install -y mongodb
 # 默认使用sqlite3，推荐使用PostgreSQL，需要配置（可以不安装）
 #apt-get install -y postgresql libpq-dev
 apt-get install -y volatility
 apt-get install -y swig
 #Tcpdump
 apt-get install -y tcpdump apparmor-utils libcap2-bin
 aa-disable /usr/sbin/tcpdump
 setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
 #特征扫描
 pip install yara-python==3.5.0
 #cuckoo 独立python env环境
 pip install virtualenv
 #测试，能导入，不报错就可以了
 #python import yara
```

#### 宿主机环境配置

```bash
#KVM 硬盘创建
#-f 硬盘格式 存放位置 大小
qemu-img create -f qcow2 /data/vm/win7x64.qcow2 20G
#硬盘格式转换
#qemu-img convert -f raw -O qcow2 test.raw test.raw.qcow2

#网络配置创建
virsh net-define /etc/libvirt/qemu/networks/default.xml #添加网络配置
virsh net-start default # 启动网络配置
virsh net-destroy default  # 删除网络配置
virsh net-undefine default # 取消网络配置
service libvirtd restart  # 重新启动kvm网络管理服务

#常用命令
virsh list --all # 显示所有虚拟机
virsh dumpxml vm # 配置文件查看
virsh edit vm # 配置文件编辑 ctrl +o ,ctrl + x 保存nano编辑方式

virsh suspend vm # 挂起
virsh resume vm # 恢复
virsh start vm # 启动
virsh reboot vm # 重启
virsh shutdown vm # 正常关机
virsh destroy vm # 强制关机
virsh undefine vm # 删除名称为vm的虚拟机
virsh autostart vm # 自启动
virsh autostart --disable vm # 关闭自启动

qemu-img snapshot -c #创建快照 快照名 磁盘路径
qemu-img snapshot -l #显示所有快照  磁盘路径
qemu-img snapshot -a #恢复到指定快照 快照名 磁盘路径
qemu-img snapshot -d #删除快照 快照名 磁盘路径

net-autostart                  自动开始网络
net-create                     从一个 XML 文件创建一个网络
net-define                     定义一个永久网络或修改一个xml文件中定义的持久网络
net-destroy                    销毁（停止）网络
net-dhcp-leases                打印给定网络的租赁信息
net-dumpxml                    XML 中的网络信息
net-edit                       为网络编辑 XML 配置
net-event                      Network Events
net-info                       网络信息
net-list                       列出网络
net-name                       把一个网络UUID 转换为网络名
net-start                      开始一个(以前定义的)不活跃的网络
net-undefine                   取消（删除）定义一个永久网络
net-update                     更新现有网络配置的部分
net-uuid                       把一个网络名转换为网络UUID

#KVM虚拟机安装，运行完等待10s，ctrl + c就可以了
sudo virt-install \
--virt-type=kvm \
--name win7x64 \
--ram 1024 \
--vcpus=1 \
--os-variant=win7 \
--virt-type=kvm \
--hvm \
--cdrom=/data/image/cn_windows_7_professional_with_sp1_x64_dvd_u_677031.iso \
--network=bridge:virbr0,model=virtio \
--graphics vnc \
--disk path=/data/vm/win7x64.qcow2
# 安装完以后一定要通过virsh edit win7x64配置 network,之后一定先关机再启动，直接重启无效
<interface type='bridge'>
    <mac address='52:54:00:73:ce:69'/>
    <source bridge='virbr0'/>
    <model type='virtio'/>
	<address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
#改为
<interface type='bridge'>
    <mac address='52:54:00:73:ce:69'/>
    <source bridge='virbr0'/>
    <model type='e1000'/>
	<address type='pci' domain='0x0000' bus='0x00' slot='0x03' function='0x0'/>
</interface>
```
##### 虚拟机配置
```bash
#虚拟机配置  -- 作.用命令行安装的，起来以后kvm的vnc管理地址是127.0.0.1:5900
ssh -L 9000:192.168.0.x:9000 cuckoo@192.168.0.x -N

#宿主机随便找个vnc连接的软件，我用的chrome插件
#正式虚拟机配置

#安装 Python 2.7
#安装 Java 8
#安装 PIL(Python截屏库)
#安装 Chrome、pdf、winrar、Firefox、office 2003等

#关闭 Windows更新
#关闭 防火墙

#配置静态IP 192.168.56.0/24 

#配置自动登陆  <USERNAME> <PSSWORD> 填自己的
#control userpasswords2
reg add "hklm\software\Microsoft\Windows NT\CurrentVersion\WinLogon" /v DefaultUserName /d <USERNAME> /t REG_SZ /f
reg add "hklm\software\Microsoft\Windows NT\CurrentVersion\WinLogon" /v DefaultPassword /d <PASSWORD> /t REG_SZ /f
reg add "hklm\software\Microsoft\Windows NT\CurrentVersion\WinLogon" /v AutoAdminLogon /d 1 /t REG_SZ /f
reg add "hklm\system\CurrentControlSet\Control\TerminalServer" /v AllowRemoteRPC /d 0x01 /t REG_DWORD /f
reg add "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" /v LocalAccountTokenFilterPolicy /d 0x01 /t REG_DWORD /f

# 开启guest用户
net user guest /active:yes

# 将/home/cuckoo/.cuckoo/agent/agent.py 拷贝至虚拟机启动项 C:\Documents and Settings\All Users\「开始」菜单\程序\启动\agent.pyw
# agent.py 改为agent.pyw  据说会无窗口，爱改不改
# python  agent.py , netstat -ano | findstr 8000  #有就ok了

#网络配置
iptables -t nat -A POSTROUTING -o ens33 -s 192.168.122.0/24 -j MASQUERADE
iptables -P FORWARD DROP
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -s 192.168.122.0/24 -j ACCEPT
iptables -A FORWARD -s 192.168.122.0/24 -d 192.168.122.0/24 -j ACCEPT
iptables -A FORWARD -j LOG 
# 主机转发开启
sysctl -w net.ipv4.ip_forward=1
sysctl -p /etc/sysctl.conf

# 打快照
qemu-img snapshot -c cuckoo /data/vm/win7x64.qcow2
````

#### Cuckoo配置


```bash
#配置默认虚拟机为kvm
vim /home/cuckoo/.cuckoo/conf/cuckoo.conf
#machinery = virtualbox 改为
#machinery = kvm
. /data/pyenv/bin/active

#新窗口
#执行失败的话，安装这个 pip install libvirt-python
(pyenv)$ cuckoo -d
#打开cuckoo自带web服务，通过主机ip访问
(pyenv)$ cuckoo web runserver 0.0.0.0:80

#上传文件测试无误后，修改配置文件conf/cuckoo.conf 
#后台运行cuckoo
#process_results = off
```



## 可能会踩到的一些坑

1. 想要通过virsh shutdown关机，***必须在虚拟机中安装acpid acpid-sysvinit，不是宿主机！***

2. 宿主机，如果不是对linux特别了解，不要采用mini版或server无桌面版

3. 虚拟机，软件安装，网络配置，全部搞完了再打快照【ps：本人打了无数个快照，这样不好】

4. cuckoo配置，cuckoo很强大，一般不会出问题，一定要仔细检测配置，出错了大多数都是配置文件有问题

5. 以上所有的宿主机命令，***不要用ROOT用户执行,用sudo命令执行***

## 参考资料

搭建参考：

<http://www.linux-kvm.org/page/Documents>

史上最强说明（有能力就按照官方文档走吧）：

<https://cuckoo.sh/docs/installation/index.html>

旧软件安装

<http://www.oldapps.com/> 

# 转载请注明来源：mu0gua.github.io