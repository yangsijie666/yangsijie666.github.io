--- 
layout: post
title: openstack学习之devstack安装
subtitle: Rocky版本安装
date: 2018-09-12
author: YangSijie
header-img: img/post-bg-alibaba.jpg
catalog: true
tags:
    - openstack
---

# devstack安装R版

官方文档:https://docs.openstack.org/devstack/latest/

### 0.CentOS系统安装要求
- 测试安装使用的是CentOS7.2
- 虚拟机最低配置2核心6GB内存
- 虚拟机作为host需要打开intel-VTX
- 需要安装图形化界面以运行PyCharm
- 语言建议选用英文以避免字符问题

### 1.安装图形化界面（centos是最小安装）

```bash
yum install -y @gnome
yum install -y @X11
systemctl set-default runlevel5
```

### 2.禁用SELinux

```bash
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

### 3.设定主机名，配置/etc/hosts文件。注意修改其中的服务器IP地址

```bash
hostnamectl set-hostname rocky.localdomain
echo -e '192.168.145.141\trocky rocky.localdomain' >> /etc/hosts
```

### 4.安装依赖包

```bash
yum -y install git deltarpm
```

### 5.安装源并修改配置以提高安装效率(可选)

```bash
yum -y install epel-release https://rdoproject.org/repos/rdo-release.rpm
sed -i 's/mirror.centos.org/mirrors.ustc.edu.cn/g' /etc/yum.repos.d/rdo*.repo
sed -i 's+download.fedoraproject.org/pub+mirrors.ustc.edu.cn+' /etc/yum.repos.d/epel.repo
sed -i 's/^#baseurl/baseurl/' /etc/yum.repos.d/epel.repo
sed -i 's/^metalink/#metalink/' /etc/yum.repos.d/epel.repo
sed -i 's/^mirrorlist/#mirrorlist/' /etc/yum.repos.d/epel.repo
```

### 6.禁用会干扰OpenStack的服务并重启系统

```bash
systemctl disable firewalld NetworkManager
reboot
```

### 7.创建stack用户

```bash
useradd -s /bin/bash -d /opt/stack -m stack
echo "stack ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/stack
```

### 8.配置root用户PIP源以提高安装效率(可选)

```bash
mkdir ~/.pip/
cat << EOF > ~/.pip/pip.conf
[global]
index-url = https://mirrors.ustc.edu.cn/pypi/web/simple
format = columns
EOF
```

### 9.配置stack用户pip源以提高安装效率(可选)

```bash
su - stack
mkdir ~/.pip/
cat << EOF > ~/.pip/pip.conf
[global]
index-url = https://mirrors.ustc.edu.cn/pypi/web/simple
format = columns
EOF
```

### 10.stack用户GIT获取指定分支的项目

```bash
cd ~
git clone http://git.trystack.cn/openstack-dev/devstack -b stable/rocky
git clone http://git.trystack.cn/openstack/requirements -b stable/rocky
```

### 11.修改脚本,避免文件下载失败导致安装中断

```bash
sed -i 's/^    PYPI_ALTERNATIVE_URL=/    sudo yum -y install python2-pip #PYPI_ALTERNATIVE_URL=/' ~/devstack/stack.sh
```

### 12.升级setuptools

```bash
sudo pip install --upgrade setuptools
```

### 13.stack用户创建local.conf文件,注意修改其中的HOST_IP变量为服务器的IP地址

```bash
cat << EOF > ~/devstack/local.conf
[[local|localrc]]
HOST_IP=192.168.145.141
SERVICE_IP_VERSION=4
DATABASE_PASSWORD=password
RABBIT_PASSWORD=password
SERVICE_TOKEN=password
SERVICE_PASSWORD=password
ADMIN_PASSWORD=password
WSGI_MODE=mod_wsgi
NOVA_USE_MOD_WSGI=False
CINDER_USE_MOD_WSGI=False
TARGET_BRANCH=stable/rocky
DOWNLOAD_DEFAULT_IMAGES=False
NEUTRON_CREATE_INITIAL_NETWORKS=False
disable_service tempest
GIT_BASE=http://git.trystack.cn
NOVNC_REPO=http://git.trystack.cn/kanaka/noVNC.git
enable_plugin zun http://git.trystack.cn/openstack/zun stable/rocky
enable_plugin zun-tempest-plugin http://git.trystack.cn/openstack/zun-tempest-plugin
#This below plugin enables installation of container engine on Devstack.
#The default container engine is Docker
enable_plugin devstack-plugin-container http://git.trystack.cn/openstack/devstack-plugin-container stable/rocky
# In Kuryr, KURYR_CAPABILITY_SCOPE is ‘local’ by default,
# but we must change it to ‘global’ in the multinode scenario.
KURYR_CAPABILITY_SCOPE=local
KURYR_ETCD_PORT=2379
enable_plugin kuryr-libnetwork http://git.trystack.cn/openstack/kuryr-libnetwork stable/rocky
# install python-zunclient from git
#LIBS_FROM_GIT="python-zunclient"
# Optional:  uncomment to enable the Zun UI plugin in Horizon
enable_plugin zun-ui http://git.trystack.cn/openstack/zun-ui stable/rocky
EOF
```

### 14.stack用户运行stack.sh,安装失败请排除错误继续运行~/devstack/stack.sh

#### 报错指引：

1. 如果遇到`env: ‘/opt/stack/requirements/.venv/bin/pip’: No such file or directory`

   则在stack用户下：

   ```bash
   [stack@kuber-node1 devstack]$ virtualenv ../requirements/.venv/
   ```

2. 如果遇到文件下载不下来，比如https://github.com/coreos/etcd/releases/download/v3.1.10/etcd-v3.1.10-linux-amd64.tar.gz，可以在自行通过wget等下载至/opt/stack/devstack/files文件夹下后，重新安装（一般直接再装就可以了）

3. 如果遇到pip源的版本找不到，请自行根据报错提示修改/opt/stack/requirements/upper-constraints.txt的相应内容

4. 如果出现SyntaxError: '<' operator not allowed in environment markers,需要安装sudo pip install --upgrade setuptools

5. 如果是pip源里的pbr版本不是最新版造成问题，要自行安装

  ```bash
  git clone http://git.trystack.cn/openstack-dev/pbr
  cd pbr
  sudo pip install
  ```

6. 如果仍然报错,请先执行如下脚本

  ```bash
  ~/devstack/unstack.sh
  ```

  卸载后，必须在local.conf中增加enable_service placement-api

### 15.安装完成启用WEB服务,停止并禁用iptables服务

```bash
sudo systemctl enable httpd
sudo systemctl stop iptables
sudo systemctl disable iptables
```

### 16.登录dashboard时可能会有报错,刷新页面即可

如果部署机重启需要重新增加虚拟磁盘供cinder使用

```bash
losetup /dev/loop0 /opt/stack/data/stack-volumes-default-backing-file
losetup /dev/loop1 /opt/stack/data/stack-volumes-lvmdriver-1-backing-file
```

重启后记得重启kuryr，现在版本有bug

```bash
systemctl restart devstack@kuryr-libnetwork
```

### 17.命令行测试

```bash
source /opt/stack/devstack/openrc admin admin
zun list
```

查看是否有输出，如果出现http406，降低zun client的版本

```bash
sudo pip install python-zunclient==1.2.0
```

httpd的服务重启（不然horizon会报错）

```bash
systemctl restart httpd
```

使用docker hub下载

```bash
zun run --name test cirros ping -c 4 8.8.8.8
```

使用glance测试

```bash
docker pull cirros
docker save cirros | openstack image create docker-test --public --container-format docker --disk-format raw
```

事先添加网络后

```bash
zun run --image-driver glance docker-test ping 127.0.0.1
```

物理主机寻找命令

```bash
nova-manage cell_v2 discover_hosts
```

### 18.安装PyCharm

```bash
su - root
cd /opt/
wget https://download.jetbrains.com/python/pycharm-community-2017.3.3.tar.gz
tar -zxf pycharm-community-2017.3.3.tar.gz
cd ~
ln -s /opt/pycharm-community-2017.3.3/bin/pycharm.sh ~/pycharm.sh 
```

### 19.root用户登录Gnome桌面,打开terminal执行命令

```bash
~/pycharm.sh 
```

### PyCharm配置debug环境详见文档

#### how-to-debug-zun-with-pycharm.docx

#### how-to-debug-zun-api.txt

### 云主机测试专用镜像

https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img



新版devstack查看日志命令：

```bash
sudo journalctl -a -f --unit devstack@zun-api
```

