---
title: 透過 Ansible 快速部署 OpenShift 3.11(OKD) 叢集
toc: true
date: 2019-08-12 23:41:13
categories: 
- Openshift
tags: 
- openshift
- ansible
thumbnail: /images/openshift 3.11.jpg
---

本篇將介紹透過 [OpenShift Ansible](https://github.com/openshift/openshift-ansible) 來快速部署 RedHat OpenShift 社區版本 `OKD (The Origin Community Distribution of Kubernetes)` 的 `release-3.11` 版本，此篇文章主要是用來筆記學習裸機機器(Bare-metal Server)部署過程與環境架構。

共同編輯: [Hank Lin - 靠著13%技術與87%嘴砲](https://chlin62.github.io/okd-311-deploy-cluster/)

<!-- more -->

## 實驗節點資訊

本次安裝作業系統採用 **CentOS 7**，測試環境為實體主機：

| IP Address| Hostname | Sepc | Remark | Hard Disk Drive|
| --------  | -------- | -------- | -------- |--------|
| 192.168.101.130 | paas01 | 8 core/ 8G| Master node| SSD 256 GB|
| 192.168.101.131 | paas02 | 8 core/ 8G| Infra node| HDD 1TB |
| 192.168.101.132 | paas03 | 8 core/ 8G| AP node| HDD 1TB |
| 192.168.101.133 | paas04 | 8 core/ 8G| AP node| HDD 500 GB |
| 192.168.101.134 | paas05 | 8 core/ 8G| AP node| HDD 500 GB |

> ISO: Centos-7-x86_64-Minimal-1810.iso

## 實驗-部署節點架構圖

### OKD: System Architecture
* 這次部署的架構主要是針對 `small size` 的 Single Master and Infra node 的架構進行部署
* Production 環境建議採用 HA 架構的 OKD cluster
* 所有進出 Cluster 流量會透過 Infra Node 進行流量進出，Web based (http/https and ws/wss) 會透過 Infra node 中的 HAProxy 進出，TCP based 的則透過 External IP

```rust

  Route(Ingress Controller)            +---------------------+
  *.apps.paas.domain.tw                |    Master node      |
    +----------------+                 |---------------------| Webconsole:
    |                |                 |                     | https://webconsole.paas.domain.tw
    |    Internet    |                 |    192.168.101.130  |
    |                |                 |                     | Cluster Console:
    +-----^----+-----+                 |paas01.paas.domain.tw| https://console.apps.paas.domain.tw
          |    |                       |     (8C/8G, SSD)    |
          |    |                       +---------+-----------+
          |    |                                 |
          |    | +---------------------+---------+-----------+----------------------+
          |    | |                     |                     |                      |
       +--+----v-+-----------+ +---------+-----------+ +---------+-----------+ +----------+----------+
       |     Infra   node    | |      AP    node     | |      AP   node      | |      AP   node      |
       |---------------------| |---------------------| |---------------------| |---------------------|
       |                     | |                     | |                     | |                     |
       |   192.168.101.131   | |    192.168.101.132  | |   192.168.101.133   | |    192.168.101.134  |
       |                     | |                     | |                     | |                     |
       |paas02.paas.domain.tw| |paas03.paas.domain.tw| |paas04.paas.domain.tw| |paas05.paas.domain.tw|
       |       (8C/8G)       | |       (8C/8G)       | |        (8C/8G)      | |        (8C/8G)      |
       +---------------------+ +---------------------+ +---------------------+ +---------------------+

```

### Master Node
* Master node 主要包含了三個 Core Componet (Control plane)
    * **ETCD**: 用於保存cluster狀態及配置
    * **API server**: Kubernetes API service的驗證以及pod配置。另外也掌管 pod scheduling and 訊息同步
    * **Conroller Manager Server**: Controller Manager 主要透過ETCD上的資料對cluster 物件進行 replication controll，並透過 API 強制將物件執行至指定狀態
* Control plane 在 OKD 3.10 之後會透過 static pod 的形式存在於 Master 中，它們的設定 yaml 檔會存在於 /etc/origin/node/pods 中，有必要時可以對其設定進行調整

```rust
                                 +-------------------------+
                                 | Control Plan Static Pod |
                                 | +---------------------+ |
    +---------------------+   +----+        ETCD         | |
    |     Master node     |   |  | +---------------------+ |
    |---------------------|   |  |                         |
    |                     |   |  | +---------------------+ |
    |    192.168.101.130  +---+----+     API Server      | |
    |                     |   |  | +---------------------+ |
    |paas01.paas.domain.tw|   |  |                         |
    |     (8C/8G, SSD)    |   |  | +---------------------+ |
    +---------------------+   +----+  Controller Server  | |
                                 | +---------------------+ |
                                 +-------------------------+
```

### Infra Node
* Infra node目前規劃上主要會存在下列幾個Componets
    * **Route (HAProxy)**: HAProxy 主要負責 HTTP/HTTPS based 的網路流量，Openshift 預設採用 HAProxy。 如果有 Nginx 需要也可透過 Nginx 將 HAProxy 置換成 Nginx ingress controller.
    * **Openshift Monitor**: 主要會透過 Cluster monitoring operator 去部署 prometheus and grafana
    * **Metrics**: kube-state-metrics，會負責監控 cluster metrics 並提供HPA 功能
* Production 環境建議infra node 分拆角色
    * Ingress, Egress, External IP 相關 service 合併為一個 routing node 只負責流量控管
    * Metrics Monitoring 相關 service 獨立出一個 Monitor node
    * logging EFK 自己有自己的 logging node

    - Remark: 因為resource不足的緣故，故沒有安裝EFK

```rust
                                  +-------------------------+
                                  |          Pod            |
                                  | +---------------------+ |
     +---------------------+   +----+    Route(HAProxy)   | |
     |    Infra  node      |   |  | +---------------------+ |
     |---------------------|   |  |                         |
     |                     |   |  | +---------------------+ |
     |    192.168.101.131  +---+----+  Openshift Monitor  | |
     |                     |   |  | +---------------------+ |
     |paas02.paas.domain.tw|   |  |                         |
     |        (8C/8G)      |   |  | +---------------------+ |
     +---------------------+   +----+    Metrics Server   | |
                                  | +---------------------+ |
                                  +-------------------------+
```


## 事前準備 

安裝前需要確認以下幾個項目：

- 所有節點的網路之間可以互相溝通。
- 部署節點對其他節點不需要 SSH 密碼即可登入。
- 所有節點都擁有 `root` 權限，並且不需要輸入密碼。
- 所有節點需要安裝 Python，CentOS 預設應該都會裝。
- 所有節點需要設定 /etc/hosts 解析到所有主機。
- 部署節點(Basion主機) 需要安裝 Ansible。

> 建議部署這邊可以透過一台在同網域的 `Basion主機` 直接操作連線至多台主機環境

### 設置主機名

為方便識別裸機機器對應名稱，這邊修改每台實體機 `hostname`

```bash=
#  Master
$　hostnamectl set-hostname paas01.paas.domain.tw
#  Infra node
$　hostnamectl set-hostname paas02.paas.domain.tw
#  AP node
$　hostnamectl set-hostname paas01.paas.domain.tw
$　hostnamectl set-hostname paas01.paas.domain.tw
$　hostnamectl set-hostname paas01.paas.domain.tw
```

### Network Setting 並設置 「DNS Server」

配置 CentOS 系統網路，透過CLI設定配置當前實體機網卡(ifcfg-xxx):

```shell=
# ifcfg-em1 是網卡名稱，如果是 eth0 修改對應網卡 ifcfg-eth0

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
IPADDR=192.168.101.XXX　　　　
PREFIX=24
GATEWAY=192.168.101.254
DNS1=8.8.8.8
IPV6_PRIVACY=no
DOMAIN=paas.domain.tw
```
### 設定完後重啟 NetworkManager

```bash=
$ systemctl restart NetworkManager
$ cat /etc/resolv.conf 

search paas.domain.tw # restart後會增加cluster.local
nameserver 8.8.8.8
```

### Bastion node 設定 (節點設置免 SSH 密碼)

* 在部署環境建議透過 Bastion 的方式進行 Cluster部署，Bastion 可以是一台Notebook 也可以是一台固定的機器，最主要我們會透過Bastion機對整個 Cluster進行部署及操作
* 在接下來的部署，我們會統一在 Bastion 機上面進行 Inventory 撰寫以及透過ansible 建置 Openshift
* 在撒 ansible playbook 前，因為 ansible 會透過 ssh 對 cluster machine 進行指令操作，所以我們要先對 cluster nodes 進行 ssh 免密碼登入設定


```shell=
$ ssh-keygen 
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:YWid3TtIfGyFM8MsaZf1YYp9RcgC60plBJrhPNqEEDk root@basion.paas.domain.tw
The key's randomart image is:
+---[RSA 2048]----+
|  oo  . ..o= =o=o|
|  E. + * =++&.=.o|
|   .. X =.B+=B ..|
|     = o * + ..  |
|    . . S o o    |
|       . .   .   |
|        .        |
|                 |
|                 |
+----[SHA256]-----+

...
...
```

```shell=
$ for seq in {1..5};
do
    ssh-copy-id root@paas0$seq.paas.domain.tw; \
    echo paas0$seq;\
done

#test ssh connection

$ for seq in {1..5};
do
    ssh root@paas0$seq.paas.domain.tw echo paas0$seq; \
done
```

### 檢查 SELinux 是否開啟 (Enforcing SELinux)

要安裝 OpenShift 或是 OKD，都必須 啟用 SELinux，否則安裝會失敗。必須設定 SELINUX=enforcing 與 SELINUXTYPE=targeted。

修改方式如下：

```shell=
$ cat /etc/selinux/config
$ enforcing
$ seenforcing
$ selinuxenabled
$ sestatus


$ cat /etc/selinux/config

# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=enforcing
# SELINUXTYPE= can take one of three values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted

---


$ for seq in {1..5};
do
    ssh root@paas0$seq.paas.domain.tw sestatus; \
    echo paas0$seq;\
done
```


## 配置 DNS 設定

此次實驗請於 Domain 提供平台，設置「區域檔紀錄」類型 A 配置每個節點如下配置：


| IP Address| Hostname | Type |  TTL |
| --------  | -------- | -------- | -------- |
| 192.168.101.130 | paas01.paas.domain.tw | A | 3600 |
| 192.168.101.131 | paas02.paas.domain.tw | A | 3600 |
| 192.168.101.132 | paas03.paas.domain.tw | A | 3600 |
| 192.168.101.133 | paas04.paas.domain.tw | A | 3600 |
| 192.168.101.134 | paas05.paas.domain.tw | A | 3600 |
| 192.168.101.131 | *.apps.paas.domain.tw | A | 3600 |
| 192.168.101.130 | webconsole.paas.domain.tw | A | 3600 |

**remark**: 部分DNS service 有提供設定wildcard domain 的服務，如果沒有提供的話，建議之後在上面透過Route 進出的apps，可以透過正向表列方式將domain指向 infra node

## 安裝依賴套件 (Install dependency packages) 

```shell=
$ for seq in {1..5};
do
    ssh root@paas0$seq.paas.domain.tw  yum install wget git net-tools bind-utils iptables-services bridge-utils bash-completion kexec-tools sos psacct bash-completion.noarch bash-completion-extras.noarch python-passlib NetworkManager -y
    echo paas0$seq;\
done
```

### 安裝 Docker (Install docker)

```shell=
$ for seq in {1..5};
do
ssh root@paas0$seq.paas.domain.tw yum install docker-1.13.1 -y;
ssh root@paas0$seq.paas.domain.tw echo pupaas0$seq done;
done


$ for seq in {1..5};
do
ssh root@paas0$seq.paas.domain.tw systemctl enable docker;
ssh root@paas0$seq.paas.domain.tw systemctl restart docker ;
ssh root@paas0$seq.paas.domain.tw echo pupaas0$seq done;
done

$ for seq in {1..5};
do
ssh root@paas0$seq.paas.domain.tw docker info | grep Version;
ssh root@paas0$seq.paas.domain.tw echo pupaas0$seq done;
done
```

## 安裝 Ansible 

此部分於部署節點 (此實驗採用 `Master節點`） 啟用 EPEL 倉庫以安裝 Ansible

```shell=
# 全局禁用EPEL套件庫，以便在安裝的後續步驟中不會有狀況
$ yum -y install epel-release;
$ yum -y --enablerepo=epel install ansible pyOpenSSL;
```

## Openshift Ansible 下載

```shell=
$ wget https://github.com/openshift/openshift-ansible/archive/openshift-ansible-3.11.136-1.zip
$ yum install unzip
$ unzip openshift-ansible-3.11.136-1.zip
$ cd openshift-ansible-openshift-ansible-3.11.136-1/
...
$ cd ..
$ cp openshift-ansible-openshift-ansible-3.11.136-1/ansible.cfg .
```

## 配置 Openshift Ansible 「inventory file」 描述

## Inventory
* Inventory 主要是設定整個Cluster 的重要角色
* 這次的 Inventory 我們主要有幾個重點
    * **Core Settings**: 主要 core componets 的設定
    * **Container Runtime Setting**: 這次我們主要會將Container runtime 轉換為RedHat 的另一個 Container runtime 專案  [CRIO](https://cri-o.io/)
    * **CA Expired Date Setting**: 將憑證日期延長至20年
    * 因為部分的 add-on 有 persistent volume 需要(e.g. EFK)，其餘設定都暫時先用設定，夠過後續安裝的方式安裝。

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

debug_level=2
##
#Core Settings
#-----------------------------------------------------
openshift_image_tag="v3.11.0"
openshift_pkg_version="-3.11.0-1.el7.git.0.62803d0"
openshift_version="3.11.0"
openshift_release="3.11.0"
openshift_master_default_subdomain=apps.paas.domain.tw
openshift_deployment_type=origin
openshift_hosted_infra_selector=""
openshift_master_cluster_hostname=webconsole.paas.domain.tw
openshift_master_cluster_public_hostname=webconsole.paas.domain.tw
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]
openshift_master_console_port=8443
openshift_master_api_port=8443
#-----------------------------------------------------

#Container runtime setting
#-----------------------------------------------------
#if you wanna change the crio
# https://docs.openshift.com/container-platform/3.11/crio/crio_runtime.html
openshift_use_crio=True
openshift_use_crio_only=False
openshift_crio_enable_docker_gc=False
openshift_crio_docker_gc_node_selector={'runtime': 'cri-o'}
#-----------------------------------------------------

#CA expired date setting
#-----------------------------------------------------
openshift_ca_cert_expire_days=7300
openshift_node_cert_expire_days=7300
openshift_master_cert_expire_days=7300
etcd_ca_default_days=7300
#-----------------------------------------------------


osm_use_cockpit=true
osm_cockpit_plugins=['cockpit-kubernetes']

## # host group for masters
[masters]
paas01.paas.domain.tw

## # host group for etcd
[etcd]
paas01.paas.domain.tw

## # host group for nodes
[nodes]
paas01.paas.domain.tw openshift_node_group_name='node-config-master-crio'
paas02.paas.domain.tw openshift_node_group_name='node-config-infra-crio'
paas03.paas.domain.tw openshift_node_group_name='node-config-compute-crio'
paas04.paas.domain.tw openshift_node_group_name='node-config-compute-crio'
paas05.paas.domain.tw openshift_node_group_name='node-config-compute-crio'
```

## Prerequisites
* 安裝 pre requisites rpm
* production mode 建議透過offline install 的方式安裝，保存安裝時所使用的rpm檔
```bash=
$ ansible-playbook -i inventory.ini openshift-ansible-openshift-ansible-3.11.136-1/playbooks/prerequisites.yml 
```

## Deploy Cluster

* 開始部署節點
```bash=
$ ansible-playbook -i inventory.ini openshift-ansible-openshift-ansible-3.11.136-1/playbooks/deploy_cluster.yml 
```


## 後續設定
- 增加 admin 帳號 (登入webconsole)
- inventory 一開始預設有指定透過/etc/origin/master/htpasswd 作為基本的 identity provider 驗證
```bash=
$ htpasswd -c /etc/origin/master/htpasswd admin
```
* 對 htpasswd admin user 進行cluster role binding (cluster-admin)
```bash=
$ oc adm policy add-cluster-role-to-user cluster-admin admin
```
* 最後我們就可以透過inventory 設定的url登入Openshift webconsole 介面了
```
Webconsole: https://webconsole.paas.domain.tw:8443
Cluster console: https://console.apps.domain.tw
```
