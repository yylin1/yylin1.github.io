---
title: 透過 Microk8s 快速部署 kubeflow
toc: true
date: 2018-09-17 12:12:02
categories: 
- Kubeflow
tags: 
- kubernetes
- microk8s
- kubeflow
- deploy
thumbnail: /images/microk8s.png
---

此文章將記錄透過安裝 Microk8s ，在本地機器上輕鬆部署 kubernetes 集群，並且透過運行腳本讓 kubeflow 環境一起部署完成。最後您將擁有簡易的單節點 Kubernetes 集群以及在 Pod 中部署為服務的 Kubeflow 的所有默認核心組件。並且能訪問 JupyterHub 筆記本和 Kubeflow Dashboard，進行相關測試。

<!-- more -->
## 什麼是microk8s

簡單來說，[microk8s](https://microk8s.io/)設計為快速輕巧Kubernetes最新版本安裝與主機隔離但不通過虛擬機。通過在單個快照包[snap](https://snapcraft.io/)中打包 Kubernetes、Docker.io、iptables 和 CNI的所有上游二進製文件來實現此隔離。[snap](https://snapcraft.io/)是一個應用程序容器，您可以將其想像為Docker容器的輕量級版本。它使用了許多相同的底層技術進行隔離，而不會有網絡隔離的所帶來的開銷。而[minikube](https://github.com/kubernetes/minikube)需使用虛擬化工具來環境隔離創建K8S集群，想必會快照更加耗時。

## 安裝Multipass 

Mac OS X. (本篇文章以 Mac 環境進行))
- 本機Mac OS 安裝程序安裝[Multipass](https://github.com/CanonicalLtd/multipass/releases/download/2018.6.1/multipass-2018.6.1-full-Darwin-signed.zip)

Linux OS
- 透過[snap](https://snapcraft.io/)使用指令安裝

```bash=
$ sudo snap install multipass --beta --classic
```

---
## 啟動Ubuntu虛擬機

部署前要先下載cloud-init文件

```bash=
wget https://bit.ly/2tOfMUA -O kubeflow.init
```
這邊查看一下文件內容，主要是配置kubeflow環境並去執行kubeflow 已經寫好script能更快速部署，有需要修正自己的環境可以修改後執行

```yaml=
# cloud-init for kubeflow

package_update: true
package_upgrade: true

runcmd:
 - mkdir /kubeflow
 - wget https://bit.ly/2tp2aOo -O /kubeflow/install-kubeflow-pre-micro.sh
 - chmod a+x /kubeflow/install-kubeflow-pre-micro.sh
 - wget https://bit.ly/2tndL0g -O /kubeflow/install-kubeflow.sh
 - chmod a+x /kubeflow/install-kubeflow.sh
 - printf "\n\nexport KUBECONFIG=/snap/microk8s/current/client.config\n\n" >> /home/multipass/.bashrc
```

### 啟動Multipass VM

```shell=
$ multipass launch bionic -n kubeflow -m 8G -d 40G -c 4 --cloud-init kubeflow.init

Retrieving image: 66%
Retrieving image: 87%
Verifying image:  -
Launched: kubeflow
```
> 預設備至 Multipass 為 Kubeflow 部署創建的 VM 上的`最低建議設置`，這邊是可以自由地調整根據主機的能力和工作負載需求。

依據環境狀況應該很快速就建立好單節點集群，這邊可以查看一下
```bash=
$ multipass list                                    
Name                    State       IPv4             Release
kubeflow                RUNNING     192.168.64.2     Ubuntu 18.04 LTS
```
> 如果部署有問題隨時都可以透過`multipass -h`查看 

接下來就mount本地文件與multipass配置
```bash=
$ multipass mount . kubeflow:/multipass
```

### 安裝kubernetes

接下來我們就可以進入vm，安裝由microk8s驅動的kubernetes，以及部署Kubeflow所需的其他工具。

透過multipass進入vm
```bash=
$ multipass shell kubeflow

Welcome to Ubuntu 18.04.1 LTS (GNU/Linux 4.15.0-34-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Sep 17 09:03:39 CST 2018

  System load:  0.63              Processes:             153
  Usage of /:   2.9% of 38.60GB   Users logged in:       0
  Memory usage: 2%                IP address for enp0s2: 192.168.64.2
  Swap usage:   0%


  Get cloud support with Ubuntu Advantage Cloud Guest:
    http://www.ubuntu.com/business/services/cloud

0 packages can be updated.
0 updates are security updates.
multipass@kubeflow:~$
```

### 安裝microk8s、etc

```bash=
$ sudo /kubeflow/install-kubeflow-pre-micro.sh

Saving to: ‘ksonnet.tar.gz’
ksonnet.tar.gz                               100%[============================================================================================>]  14.77M   934KB/s    in 44s

2018-09-17 09:22:01 (342 KB/s) - ‘ksonnet.tar.gz’ saved [15491217/15491217]

ks_0.11.0_linux_amd64/CHANGELOG.md
ks_0.11.0_linux_amd64/CODE-OF-CONDUCT.md
ks_0.11.0_linux_amd64/CONTRIBUTING.md
ks_0.11.0_linux_amd64/LICENSE
ks_0.11.0_linux_amd64/README.md
ks_0.11.0_linux_amd64/ks
Checking kube-system status until all pods are running (2 not running)
Checking kube-system status until all pods are running (3 not running)

Before running install-kubeflow.sh, please 'export GITHUB_TOKEN=<your token>'
```
> 這邊提供GitHub Token是為了避免[ksonnet](https://github.com/ksonnet/ksonnet/blob/master/docs/troubleshooting.md)部署反覆部署可能會造成GitHub限速問題，可以[參考](https://ephrain.net/mac-homebrew-%E5%87%BA%E7%8F%BE-github-api-rate-limit-exceed-%E7%9A%84%E8%A8%8A%E6%81%AF/)建立Token。

### 更換自己的Token

```bash=
$ export GITHUB_TOKEN=972702e4fb348d3dba52f5dd3b99d7ae83e857ab
```
### kubeflow 部署

在vm環境中執行kubeflow script，就可以快速完成kubeflow部署，而這邊會有`Checking kubeflow status until all pods are running (7 not running). Sleeping for 10 seconds.`狀況就依照環境是否把所有應該生成的Pod建立完成，就執行結束。

```bash=
$ multipass@kubeflow:~$ /kubeflow/install-kubeflow.sh
namespace/kubeflow created
INFO Using context "microk8s" from kubeconfig file "/snap/microk8s/current/client.config"
INFO Creating environment "default" with namespace "default", pointing to cluster at address "http://127.0.0.1:8080"
INFO Generating ksonnet-lib data at path '/home/multipass/my-kubeflow/lib/v1.11.1'
INFO Retrieved 22 files
INFO Retrieved 5 files
INFO Retrieved 5 files
·
·
·
Checking kubeflow status until all pods are running (7 not running). Sleeping for 10 seconds.

JupyterHub Port: 31808
```
以上就完成`microk8s+kubeflow`部署

---

## 部署完成後環境

完成後檢查 Kubeflow 元件部署結果：

```shell=
multipass@kubeflow:~$ kubectl -n kubeflow get po -o wide
NAME                                  READY     STATUS    RESTARTS   AGE       IP          NODE
ambassador-68954d75f4-2n6lv           2/2       Running   4          4h        10.1.1.55   kubeflow
ambassador-68954d75f4-7dgvl           2/2       Running   4          4h        10.1.1.56   kubeflow
ambassador-68954d75f4-bgs8h           2/2       Running   5          4h        10.1.1.44   kubeflow
mxnet-operator-f46557c4f-wmntn        1/1       Running   2          3h        10.1.1.52   kubeflow
spartakus-volunteer-d54b65666-klrfk   1/1       Running   2          4h        10.1.1.45   kubeflow
tf-hub-0                              1/1       Running   2          4h        10.1.1.47   kubeflow
tf-job-dashboard-784cdcbb4f-j2cjx     1/1       Running   2          4h        10.1.1.51   kubeflow
tf-job-operator-85b46d47b7-xfwmp      1/1       Running   2          4h        10.1.1.48   kubeflow
```

### 運行JupyterHub測試
透過查看vm狀態中，`完成後連接 http://IP:Port`，並輸入`任意帳號密碼`進行登入。

![](https://i.imgur.com/aecctnI.png)

**Spawner options**可以自行配置Jupyter Notebook 資源狀態
> Image: 預設會有多種映像檔可以使用
> 其他參數可調整 CPU、Memory、GPU資源限制

![](https://i.imgur.com/nEaOr0L.png)

按下Spawn kubeflow就會啟動一個pod配置資源給Jupyter notebook，部署過程時Image下載需要花一點時間。

![](https://i.imgur.com/Acr64W9.png)
> 配置過程中有時會後有狀況，可能需要多嘗試

完成後即可在透過Jupyter Notebook 編寫Model進行DL訓練
![](https://i.imgur.com/sCqtj53.png)

## Kubernetes Addons

microk8s在 Kubernetes 安裝了一個`準系統`。這代表只要安裝和運行 api-server、controller-manager、scheduler、kubelet，cni、kube-proxy。可以使用該 `microk8s.enable` 命令運行 kube-dns 和 Dashboard 等附加服務。

```shell=
microk8s.enable dns dashboard
```
使用該`disable`命令隨時禁用這些插件
```shell=
microk8s.disable dashboard dns
```

```bash=
$ microk8s.kubectl cluster-info
Kubernetes master is running at http://127.0.0.1:8080
Heapster is running at http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Grafana is running at http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
InfluxDB is running at http://127.0.0.1:8080/api/v1/namespaces/kube-system/services/monitoring-influxdb:http/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.
```
## 停止並重新啟動microk8 (測試環境為Mac)
> 在 Linx 環境指令都是偷過`snap` enable/disable 來管理microk8s

Mac環境可以透過`multipass -h`查詢可執行指令

### 暫時關閉vm運行
```shell=
multipass  stop kubeflow
```
### 啟動vm運行
```shell=
multipass  stop kubeflow
```
### 查看multipass目前vm狀態
```shell=
Name                    State       IPv4             Release
kubeflow                STOPPED     --               Ubuntu 18.04 LTS
```

### 刪除microk8s 
透過`multipass list` 檢查instances狀態

如果只下delete只有instances狀態刪除，是可以透過recover恢復已刪除的instances
```shell=
multipass delete kubeflow
Name                    State       IPv4             Release
kubeflow                DELETED     --               Not Available
```
所以我們要透過`purge`來清除永久清除所有已刪除的instances
```shell=
multipass purge
```


相關文章:
[[Kubernetes] A local Kubernetes with microk8s](https://itnext.io/a-local-kubernetes-with-microk8s-33ee31d1eed9)
[[kubeflow] Microk8s for Kubeflow](https://www.kubeflow.org/docs/started/getting-started-multipass/)

** 本篇文章主介紹microk8s部署kubeflow，更多相關[kubeflow](https://www.kubeflow.org/) 實作會在後續文章陸續介紹**


