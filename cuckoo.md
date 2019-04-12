# cuckoo + virtualbox

*搭建完以后敲的，有误留言，谢谢。*

------

#### 初始环境准备

> 系统版本：Ubuntu 18.4  Python：2.7 
>
#### 宿主机配置

> > > **主机依赖库**
> ``` bash
> apt-get update
> apt-get install -y python python-pip python-dev libffi-dev libssl-dev
> apt-get install -y python-virtualenv python-setuptools
> apt-get install -y libjpeg-dev zlib1g-dev swig
> # 如果使用web交互界面，需要安装mongodb，ubuntu 12.04安装mongodb有问题，不能简单安装，需要参考官网文档进行安装 （可以不安装）
> #apt-get install -y mongodb
> # 默认使用sqlite3，推荐使用PostgreSQL，需要配置（可以不安装）
> #apt-get install -y postgresql libpq-dev
> apt-get install -y volatility
> apt-get install -y swig
> #Tcpdump
> apt-get install -y tcpdump apparmor-utils libcap2-bin
> aa-disable /usr/sbin/tcpdump
> setcap cap_net_raw,cap_net_admin=eip /usr/sbin/tcpdump
> ```

> > > **python 依赖**
>
> ```bash
> #特征扫描
> pip install yara-python==3.5.0
> #cuckoo 独立python env环境
> pip install virtualenv
> 
> #测试，能导入，不报错就可以了
> python
> import yara
> ```
>
> **virtualbox**
>
> ```bash
> # virtualbox: http://download.virtualbox.org/virtualbox (我下载的6.0.0)
> dpkg -i virtualbox-xxxxxx.deb
> #上边如果出错的话，先apt-get remove virtualbox-6.0
> apt-get install virtualbox-6.0
> ```

#### 虚拟环境配置
>
>```bash
>#创建独立用户，给cuckoo和virtualbox
>adduser cuckoo
>usermod -a -G vboxusers cuckoo
>```
>
>```bash
>#cuckoo环境配置
>cd /opt
>virtualenv pyenv
>. pyenv/bin/activate
>(pyenv)$ pip install -U pip setuptools
>(pyenv)$ pip install -U cuckoo
>(pyenv)$ cuckoo -d
>```

```bash
#建议虚拟机安装时，用GUI界面，命令行吃不消...
#虚拟机安装 Win7x32 win764 winxp  msdn下载，然后用vbox安装虚拟机
#虚拟机安装   略~略~略~

#虚拟机配置  -- 作.用命令行安装的，起来以后vbox的vnc管理地址是127.0.0.1:9000
#宿主机连接vnc：-N 不连接控制台  
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

#vboxmanage hostonlyif create （第一次要先创建hostonly虚拟网卡 vboxnet0）
vboxmanage hostonlyif ipconfig vboxnet0 --ip 192.168.56.1 --netmask 255.255.255.0 
#vboxmanage hostonlyif ipconfig vboxnet0  --dhcp  dhcp也可以
vboxmanage modifyvm win7x86 --nic1 hostonly --hostonlyadapter1 vboxnet0
vboxmanage modifyvm win7x64 --nic1 hostonly --hostonlyadapter1 vboxnet0
vboxmanage modifyvm winxp --nic1 hostonly --hostonlyadapter1 vboxnet0
#做完以上主机配置，软件安装，网络配置，然后打快照
VBoxManage snapshot "win7x86" take "win7x86_snapshot" --pause
VBoxManage snapshot "win7x64" take "win7x64_snapshot" --pause
VBoxManage snapshot "winxp" take "winxp_snapshot" --pause

```

#### Cuckoo配置

1. 修改Cuckoo配置文件，路径：/home/cuckoo/.cuckoo/conf/virtualbox.conf

```bash
machines = win7x86,win7x64,winxp

[win7x86]
label = win7x86
platform = windows
ip = 192.168.56.10
snapshot = win7x86_snapshot
[win7x64]
label = win7x64
platform = windows
ip = 192.168.56.11
snapshot = win7x64_snapshot
[winxp]
label = winxp
platform = windows
ip = 192.168.56.12
snapshot = winxp_snapshot
```
2. 修改conf/reporting.conf文件，配置数据库，我用了mongodb

```bash
[mongodb]
enabled = yes
```
3. 安装特征库

```bash
(pyenv)$cuckoo community
(pyenv)$ cuckoo community --file cuckoo_master.tar.gz
```
4. 配置防火墙(抄来的)

```bash
iptables -t nat -A POSTROUTING -o eth0 -s 192.168.56.0/24 -j MASQUERADE
iptables -P FORWARD DROP
iptables -A FORWARD -m state --state RELATED,ESTABLISHED -j ACCEPT
iptables -A FORWARD -s 192.168.56.0/24 -j ACCEPT
iptables -A FORWARD -s 192.168.56.0/24 -d 192.168.56.0/24 -j ACCEPT
iptables -A FORWARD -j LOG 
# 主机转发开启
sysctl -w net.ipv4.ip_forward=1
sysctl -p /etc/sysctl.conf
```
5. 测试cuckoo

```bash
#新窗口
cuckoo -d
#打开cuckoo自带web服务，通过主机ip访问
cuckoo web runserver 0.0.0.0:80
#上传文件测试无误后，修改配置文件conf/cuckoo.conf 
#后台运行cuckoo
process_results = off

```

## 可能会踩到的一些坑

1. 环境一般都是复制，粘贴执行，就完事了，建议采用debian、Ubuntu【除12.04】，其他环境可能会有问题
2. 宿主机，如果不是对linux特别了解，不要采用mini版或server无桌面版
3. 虚拟机，软件安装，网络配置，全部搞完了再打快照【ps：本人打了无数个快照，这样不好】
4. cuckoo配置，cuckoo很强大，一般不会出问题，一定要仔细检测配置，出错了大多数都是配置文件有问题
5. 以上所有的宿主机命令，***不要用ROOT用户执行,用sudo命令执行***

## 参考资料

搭建参考：

http://www.fualan.com/article/31/

史上最强说明（有能力就按照官方文档走吧）：

<https://cuckoo.sh/docs/installation/index.html>

旧软件安装

<http://www.oldapps.com/> 

#### 
