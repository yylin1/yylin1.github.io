---
title: 透過 Ansible 快速部署安 Openshift 4.0 叢集
toc: true
date: 2019-01-13 22:44:13
categories: 
- Openshift
tags: 
- openshift
- ansible
thumbnail: /images/openshift-install.png
---

本篇將介紹透過 [OpenShift Ansible](https://github.com/openshift/openshift-ansible) 來快速部署實體主機Openshift 4.0叢集版本，這邊主要透過安裝最新版本(v4.0)依賴套件來部署裸機(Bare machines)過程。

共同編輯: [chlin62](https://github.com/chlin62)
<!-- more -->

[Openshift](https://www.openshift.com/) 由 Red Hat 公司推出的一個基於主流的容器技術 Docker 和 kubernetes 構建的的開源容器應用平台。Openshift 底層以 Docker 作為容器引擎驅動，以 Kubernetes 作為容器編排引擎組件，並提供了開發語言、中間件、DevOps自動化流程工具和 Web console 用戶介面等元素，提供了一套完整的基於容器的PaaS應用雲平台。

本篇將介紹透過 [OpenShift Ansible](https://github.com/openshift/openshift-ansible) 來快速部署實體主機 Openshift 4.0叢集版本，這邊主要透過安裝最新版本(v4.0)依賴套件來部署裸機(Bare machines)過程。


本次 Openshift 安裝版本：

- Openshift Community v4.0
    - [[OKD: The Origin Community Distribution of Kubernetes]](https://github.com/openshift/origin)


## OpenShift 部署可以分為以下幾個階段：

1、**環境事前準備**：準備OpenShift叢集需要的主機，並進行事前環境相關連結配置。

2、**安裝依賴套件**：提前安裝使用Ansible安裝OpenShift叢集所依賴的最新軟版本`.rpm`套件。

3、**Ansible執行安裝**：安裝使用Ansible Playbook進行自動化安裝。

4、**OpenShift系統配置**：在使用Ansible執行安裝完成之後的系統配置。

5、**測試與額外套件** : 測試部署後的環境與額外add-on套件於叢集環境。


## 實驗節點資訊

本次安裝作業系統採用 **CentOS 7.5**，測試環境為實體主機：

| IP Address| Hostname | Sepc | Remark| 
| --------  | -------- | -------- | -------- |
| 120.110.0.130 | pupaas01 | 8 core/ 8G| Master node|
| 120.110.0.131 | pupaas02 | 8 core/ 8G| Infra node|
| 120.110.0.132 | pupaas03 | 8 core/ 8G| AP node|



## 事前準備 

安裝前需要確認以下幾個項目：

- 所有節點的網路之間可以互相溝通。
- 部署節點對其他節點不需要 SSH 密碼即可登入。
- 所有節點都擁有 Sudoer 權限，並且不需要輸入密碼。
- 所有節點需要安裝 Python。
- 所有節點需要設定/etc/host解析到所有主機。
- 部署節點需要安裝 Ansible。

> 建議部署這邊可以透過一台在同網域的`basion主機`直接操作多台主機環境

---

### 設置主機名

為方便識別主機對應名稱，這邊修改每台實體機`hostname`
```bash=
# Master
$　hostnamectl set-hostname pupaas01
# Infra node
$　hostnamectl set-hostname pupaas02
# AP node
$　hostnamectl set-hostname pupaas03
```

### Network Setting 設置 Domain

安裝 CentOS Linux 7 作業系統時，網路透過CLI設定配置，而這邊需要先配置好`Openshift`訪問網頁需要配置`Domain`。

>  ifcfg-em1 是網卡名稱，如果是 eth0 修改對應網卡 ifcfg-eth0

每台節點編輯配置網路設定，並在最後面加上自己的 `DOMAIN="xxx.xx.xx"`。

```bash=
$ vim /etc/sysconfig/network-scripts/ifcfg-em1

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=em1
UUID=7c200433-ead4-43e3-a571-0dfeb0515a96
DEVICE=em1
ONBOOT=yes
IPADDR=120.110.0.XXX　　　　
PREFIX=24
GATEWAY=120.110.0.XXX
DNS1=8.8.8.8
IPV6_PRIVACY=no
DOMAIN=paas.nctu.me
```

設定完後重啟 NetworkManager
```bash=
$ systemctl restart NetworkManager
$ cat /etc/resolv.conf 

search paas.nctu.me
nameserver 8.8.8.8
```

> [Ref: [How can I add additional search domains to the resolv.conf created by dhclient in CentOS]](https://superuser.com/questions/110808/how-can-i-add-additional-search-domains-to-the-resolv-conf-created-by-dhclient-i)

### 設置 /etc/hosts 配置

為部署方便，讓節點網路之間可以互相溝通，在`/etc/hosts`中添加了節點資訊縮寫，來解析各主機。

- `所有節點`需要設定`/etc/hosts`解析到所有叢集主機。

```bash=
$ vim /etc/hosts

##
# Host Database
#
# localhost is used to configure the loopback interface
# when the system is booting.  Do not change this entry.
##
127.0.0.1       localhost
255.255.255.255 broadcasthost
::1             localhost
0.0.0.0 pubads.g.doubleclick.net
0.0.0.0 securepubads.g.doubleclick.net


120.110.0.130 pupaas01.paas.nctu.me pupaas01
120.110.0.131 pupaas02.paas.nctu.me pupaas02
120.110.0.132 pupaas03.paas.nctu.me pupaas03
```

這邊直接將`Master節點`設定配置`/etc/hosts`傳送給其他節點

```bash=
#copy to other hosts
$ for seq in {1..3};
do
    scp /etc/hosts root@pupaas0$seq:/etc/hosts
    echo pupaas0$seq
done
```


### 設置免 SSH 密碼

為方便部署節點對其他節點免密碼即可登入，這邊直接透過`Master 節點`複製`public key`給其他節點。

- [參考: 實現免密碼 ssh 登入遠端主機](http://blog.codylab.com/cmd-ssh-copy-id/)

```bash=
# basion
$ ssh-keygen -t rsa

$ for seq in {1..3};
do
    ssh-copy-id -i ~/.ssh/id_rsa.pub  root@pupaas0$seq;
    echo pupaas0$seq;
done

$ for seq in {1..3};
do
    ssh root@pupaas0$seq echo pupaas0$seq done;
done
```

### 檢查SELINUX是否開啟

修改方式如下：

```bash=
$ /etc/selinux/config

SELINUX=enforcing
SELINUXTYPE=targeted
```



### 安裝依賴套件 (Install dependency packages)

- `所有的主機`下執行以下命令安裝OpenShift依賴套件包

```bash=
$ yum install -y update

$ for seq in {1..3};
do
    ssh root@pupaas0$seq yum install wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct bash-completion.noarch bash-completion-extras.noarch python-passlib NetworkManager -y
    ssh root@pupaas0$seq echo pupaas0$seq done;
done
```
### 安裝 Docker (Install docker)

- `所有節點`都需要執行安裝docker

```bash=
$ for seq in {1..3};
do
ssh root@pupaas0$seq yum install docker-1.13.1 -y;
ssh root@pupaas0$seq echo pupaas0$seq done;
done


$ for seq in {1..3};
do
ssh root@pupaas0$seq systemctl enable docker;
ssh root@pupaas0$seq systemctl restart docker ;
ssh root@pupaas0$seq echo pupaas0$seq done;
done

$ for seq in {1..3};
do
ssh root@pupaas0$seq docker info | grep Version;
ssh root@pupaas0$seq echo pupaas0$seq done;
done
```

### 安裝本地儲存庫 (Install local repostory)

```bash=
# basion
$ yum -y install httpd
$ systemctl restart httpd
```

```bash=
# change defualt httpd port
$ nano /etc/httpd/conf/httpd.conf 
# 修改第41行 {IP}:8080
Listen 127.0.0.1:8080
```

```bash=
#testing {IP}:8080
$ curl 127.0.0.1:8080
```

###  安装依賴套件（Install openshit 4.0 alpha depency package）

這邊移動套件來源路徑

```bash=
$ cd /var/www/html/
```
- 這邊需要一個一個下載依賴套件到資料夾裡，如果安裝套件`.rpm`需要在離線環境時時，可以先透過可以連線`Internet`機器下載完後，再`scp`進去至離線環境

- 連結下載：[[Download openshift-origin-v4.0 depency package]](https://rpms.svc.ci.openshift.org/openshift-origin-v4.0/)

`master節點`安裝方式使用 `wget xxx.rpm`，或是網頁點選下載`rpm`檔後，用rpm指令安裝

```bash=
$ ls /var/www/html/

local-release.repo
origin-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
origin-clients-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
origin-docker-excluder-4.0.0-0.alpha.0.892.b6f540d.noarch.rpm
origin-excluder-4.0.0-0.alpha.0.892.b6f540d.noarch.rpm
origin-hyperkube-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
origin-hypershift-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
origin-local-release.repo
origin-master-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
origin-node-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
origin-pod-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
origin-sdn-ovs-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
origin-template-service-broker-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
origin-tests-4.0.0-0.alpha.0.892.b6f540d.x86_64.rpm
repodata
```
使用命令`createrepo`將現有的剛剛下載好的origin-4.0`.rpm`創建為自定義的yum倉庫源
```bash=
$ createrepo /var/www/html/
```

> 如果由 `Master節點` 當 basion 機時，必須要修改預設的 `firewall`，要不然node會抓不到東西

```bash=
#close firewall
$ systemctl stop firewalld

# Edit iptables
$  vim /etc/sysconfig/iptables

# 新增 rule (26行)
-A INPUT -p tcp -m state --state NEW -m tcp --dport 8080 -j ACCEPT

# 重啟firewalld
$ systemctl restart firewalld
```

### 編輯套件庫 (Edit repos）

- 這邊由`Master節點`複製 openshift 套件包傳遞給所有節點

```bash=
$ ls /etc/yum.repos.d/
CentOS-Base.repo  CentOS-OpenShift-Origin.repo  epel.repo  epel-testing.repo
```

```bash=
$ for seq in {1..3};
do
ssh root@pupaas0$seq rm -rf /etc/yum.repos.d/*;
ssh root@pupaas0$seq echo pupaas0$seq done;
done

$ for seq in {1..3};
do
scp /etc/yum.repos.d/* root@pupaas0$seq:/etc/yum.repos.d/ ;
ssh root@pupaas0$seq echo pupaas0$seq done;
done

$ for seq in {1..3};
do
ssh root@pupaas0$seq yum clean all;
ssh root@pupaas0$seq yum makecache;
ssh root@pupaas0$seq echo pupaas0$seq done;
done
```

### 確認預設 Openshift Repository 狀態

- CentOS-Base.repo

```bash=
$ cat /etc/yum.repos.d/CentOS-Base.repo

# CentOS-Base.repo
#
# The mirror system uses the connecting IP address of the client and the
# update status of each mirror to pick mirrors that are updated to and
# geographically close to the client.  You should use this for CentOS updates
# unless you are manually picking other mirrors.
#
# If the mirrorlist= does not work for you, as a fall back you can try the 
# remarked out baseurl= line instead.
#
#

[base]
name=CentOS-$releasever - Base
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#released updates 
[updates]
name=CentOS-$releasever - Updates
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=updates&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/updates/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that may be useful
[extras]
name=CentOS-$releasever - Extras
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=extras&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/extras/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7

#additional packages that extend functionality of existing packages
[centosplus]
name=CentOS-$releasever - Plus
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=centosplus&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/centosplus/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
```

- CentOS-OpenShift-Origin.repo 
```bash=
$ cat /etc/yum.repos.d/CentOS-OpenShift-Origin.repo 

[centos-openshift-origin]
name=CentOS OpenShift Origin
baseurl=http://mirror.centos.org/centos/7/paas/x86_64/openshift-origin/
enabled=1
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-PaaS

[centos-openshift-origin-testing]
name=CentOS OpenShift Origin Testing
baseurl=http://buildlogs.centos.org/centos/7/paas/x86_64/openshift-origin/
enabled=0
gpgcheck=0
gpgkey=file:///etc/pki/rpm-gpg/openshift-ansible-CentOS-SIG-PaaS

[centos-openshift-origin-debuginfo]
name=CentOS OpenShift Origin DebugInfo
baseurl=http://debuginfo.centos.org/centos/7/paas/x86_64/
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/openshift-ansible-CentOS-SIG-PaaS

[centos-openshift-origin-source]
name=CentOS OpenShift Origin Source
baseurl=http://vault.centos.org/centos/7/paas/Source/openshift-origin/
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/openshift-ansible-CentOS-SIG-PaaS

[centos-openshift-origin-source]
name=CentOS OpenShift Origin Source 4.0
baseurl=http://pupaas01:8080/
enabled=1
gpgcheck=0
```

-  epel-testing.repo
```bash=
$ cat /etc/yum.repos.d/epel-testing.repo

[epel-testing]
name=Extra Packages for Enterprise Linux 7 - Testing - $basearch
#baseurl=http://download.fedoraproject.org/pub/epel/testing/7/$basearch
metalink=https://mirrors.fedoraproject.org/metalink?repo=testing-epel7&arch=$basearch
failovermethod=priority
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

[epel-testing-debuginfo]
name=Extra Packages for Enterprise Linux 7 - Testing - $basearch - Debug
#baseurl=http://download.fedoraproject.org/pub/epel/testing/7/$basearch/debug
metalink=https://mirrors.fedoraproject.org/metalink?repo=testing-debug-epel7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1

[epel-testing-source]
name=Extra Packages for Enterprise Linux 7 - Testing - $basearch - Source
#baseurl=http://download.fedoraproject.org/pub/epel/testing/7/SRPMS
metalink=https://mirrors.fedoraproject.org/metalink?repo=testing-source-epel7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1
```

- epel.repo

```bash=
$ cat /etc/yum.repos.d/epel.repo

[epel]
name=Extra Packages for Enterprise Linux 7 - $basearch
#baseurl=http://download.fedoraproject.org/pub/epel/7/$basearch
metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-7&arch=$basearch
failovermethod=priority
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7

[epel-debuginfo]
name=Extra Packages for Enterprise Linux 7 - $basearch - Debug
#baseurl=http://download.fedoraproject.org/pub/epel/7/$basearch/debug
metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-debug-7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1

[epel-source]
name=Extra Packages for Enterprise Linux 7 - $basearch - Source
#baseurl=http://download.fedoraproject.org/pub/epel/7/SRPMS
metalink=https://mirrors.fedoraproject.org/metalink?repo=epel-source-7&arch=$basearch
failovermethod=priority
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-EPEL-7
gpgcheck=1
```

## 安裝 Ansible 

- `Master節點`啟用EPEL倉庫以安裝Ansible
```bash=
# basion
# 全局禁用EPEL套件庫，以便在安裝的後續步驟中不會有狀況
$ yum -y install epel-release;
ed -i -e "s/^enabled=1/enabled=0/" /etc/yum.repos.d/epel.repo;
$ yum -y --enablerepo=epel install ansible pyOpenSSL;
```

- 從 github 上clone openshift-ansible 4.0版本
```bash=
$ wget https://github.com/openshift/openshift-ansible/archive/openshift-ansible-4.0.0-0.137.0.zip
$ unzip openshift-ansible-4.0.0-0.137.0.zip
$ mv  openshift-ansible-openshift-ansible-4.0.0-0.137.0/ openshift-ansible-4.0
```
## 配置ansible inventory 文件

- openshift v3.11 :
```bash=
$ cat inventory.yml.311
#reate an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes
etcd
#
# # Set variables common for all OSEv3 hosts
[OSEv3:vars]
##
openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability,package_version
## # SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=root
##
openshift_image_tag="v3.11.0"
openshift_pkg_version="-3.11.0-1.el7.git.0.62803d0"
openshift_version="3.11.0"
openshift_release="3.11.0"
openshift_master_default_subdomain=apps.paas.nctu.me
#osm_cluster_network_cidr=10.128.0.0/14
#openshift_portal_net=192.168.0.0/16
#openshift_dns_ip=192.168.0.1
##
## # If ansible_ssh_user is not root, ansible_become must be set to true
##ansible_become=true
##
openshift_deployment_type=origin
deployment_subtype=registry
openshift_hosted_infra_selector=""
##
## # uncomment the following to enable htpasswd authentication; defaults to DenyAllPasswordIdentityProvider
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]

## # host group for masters
[masters]
pupaas01.paas.nctu.me

## # host group for etcd
[etcd]
pupaas01.paas.nctu.me

## # host group for nodes
[nodes]
pupaas01.paas.nctu.me openshift_node_group_name='node-config-master'
pupaas02.paas.nctu.me openshift_node_group_name='node-config-infra'
pupaas03.paas.nctu.me openshift_node_group_name='node-config-compute'
```

openshift v4.0:


```bash=
#reate an OSEv3 group that contains the masters and nodes groups
[OSEv3:children]
masters
nodes
etcd
#
# # Set variables common for all OSEv3 hosts
[OSEv3:vars]
##
openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability,package_version
## # SSH user, this user should allow ssh based auth without requiring a password
ansible_ssh_user=root
##
openshift_image_tag="v4.0.0"
openshift_pkg_version="-4.0.0-0.alpha.0.892.b6f540d"
openshift_version="4.0.0"
openshift_release="4.0.0"
openshift_master_default_subdomain=apps.paas.nctu.me
openshift_enable_service_catalog=false
#osm_cluster_network_cidr=10.128.0.0/14
#openshift_portal_net=192.168.0.0/16
#openshift_dns_ip=192.168.0.1
##
## # If ansible_ssh_user is not root, ansible_become must be set to true
##ansible_become=true
##
openshift_deployment_type=origin
deployment_subtype=registry
openshift_hosted_infra_selector=""
##
## # uncomment the following to enable htpasswd authentication; defaults to DenyAllPasswordIdentityProvider
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]

## # host group for masters
[masters]
pupaas01.paas.nctu.me

## # host group for etcd
[etcd]
pupaas01.paas.nctu.me

## # host group for nodes
[nodes]
pupaas01.paas.nctu.me openshift_node_group_name='node-config-master'
pupaas02.paas.nctu.me openshift_node_group_name='node-config-infra'
pupaas03.paas.nctu.me openshift_node_group_name='node-config-compute'
```

> 待更新中
