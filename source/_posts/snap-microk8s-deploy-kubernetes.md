---
title: 透過microk8s更輕巧建立單節點kubernetes叢集（含GPU測試）
toc: true
date: 2018-12-14 23:10:49
categories: 
- Deploy
tags:
- kubernetes
- microk8s
- kubeflow
- deploy
thumbnail: /images/microk8s-deploy-kubernetes.png
---

先前文章參考：[透過 Microk8s 快速部署 Kubeflow
](https://yylin1.github.io/2018/09/17/microk8s-deploy-kubeflow/)


此篇想重新介紹透過 microk8s，近期剛好看到[Using GPGPUs with Kubernetes](https://blog.ubuntu.com/2018/12/10/using-gpgpus-with-kubernetes?fbclid=IwAR05YY0s2aOifQQj6prD7phDyvdgXDJM_UIriMcwzsoLorni97ovVFnJ8yI)這邊文章介紹，想起之前在實驗kubeflow時候有於Mac上執行過Microk8s，這次重新測試於單機 Linux 環境並測試相關 Addons 與GPU環境中實驗。

<!-- more -->
## 什麼是 microk8s? 

```bash=
# What is microk8s? 
           _                _    ___
 _ __ ___ (_) ___ _ __ ___ | | _( _ ) ___
| '_ ` _ \| |/ __| '__/ _ \| |/ / _ \/ __|
| | | | | | | (__| | | (_) |   < (_) \__ \
|_| |_| |_|_|\___|_|  \___/|_|\_\___/|___/
```

由 Ubuntu 子公司 [Canonica](https://www.canonical.com/) 推出的 [microK8s](https://microk8s.io/)， 這是一種快速建置Kubernetes單節點的快照包，目前支援可安裝在42種Linux系統上，只要任何支持`Snap` Linux發行都可以透過 MicroK8s 快速部署。藉由小磁區和內存佔用，MicroK8s提供了一種在幾秒鐘內部署Kubernetes最新上游版本的有效解決方案，無論是在個人主機、伺服器、雲服務設備上，都能快速執行測試。

為了進一步加速Kubernetes採用並簡化常見的開發人員使用情境，MicroK8s包括越來越多的`Add-on`服務，包括容器映像檔儲存、存儲傳遞和本機GPGPU支援，所有這些服務都可直接通過單個命令啟用。對於資料科學家和機器學習工程師而言，單節點測試環境有GPGPU支持，簡化實驗過程中環境應用上相關的工作流程。

## 實驗環境

- 測試機器為Ubuntu 16.04並包含一張GTX 1060 GPU卡
- 已經事先安裝完成`CUDA Version 10.0.130`、`Nvidia Driver 410.79`


## Installing snap on Ubuntu (若環境沒有安裝)

```bash=
$ sudo apt update
$ sudo apt install snapd
```

```bash=
$ snap version
snap    2.36.1
snapd   2.36.1
series  16
ubuntu  18.04
kernel  4.15.0-39-generic
```

## 部署 microk8s


最快的入門方法是通過單擊`snap store`按鈕直接從快照存儲中安裝microk8s。

這邊還是使用CLI命令列介面安裝microk8s，並讓它更新到最新的穩定上游Kubernetes版本：

```bash=
$ sudo snap install microk8s --classic

microk8s v1.13.0 from Canonical✓ installed
```

若要指定不同kubernetes版本，我們可以在安裝過程選擇版本。例如要指定v1.12 version：

```bash=
$ sudo snap install microk8s --classic --channel=1.12/stable

microk8s (1.12/stable) v1.12.3 from Canonical✓ installed
```

可以透過`snap info microk8s`看查看當前發布的穩定版、測試版等多個版本: 

```bash=
$ snap info microk8s

name:      microk8s
summary:   Kubernetes for workstations and appliances
publisher: Canonical✓
contact:   https://github.com/ubuntu/microk8s
license:   unset
description: |
  MicroK8s is a small, fast, secure, single node Kubernetes that installs on just
  about any Linux box. Use it for offline development, prototyping, testing, or use
  it on a VM as a small, cheap, reliable k8s for CI/CD. It's also a great k8s for
  appliances - develop your IoT apps for k8s and deploy them to MicroK8s on your
  boxes.
commands:
  - microk8s.config
  - microk8s.disable
  - microk8s.docker
  - microk8s.enable
  - microk8s.inspect
  - microk8s.istioctl
  - microk8s.kubectl
  - microk8s.reset
  - microk8s.start
  - microk8s.status
  - microk8s.stop
services:
  microk8s.daemon-apiserver:          simple, enabled, active
  microk8s.daemon-apiserver-kicker:   simple, enabled, active
  microk8s.daemon-controller-manager: simple, enabled, active
  microk8s.daemon-docker:             simple, enabled, active
  microk8s.daemon-etcd:               simple, enabled, active
  microk8s.daemon-kubelet:            simple, enabled, active
  microk8s.daemon-proxy:              simple, enabled, active
  microk8s.daemon-scheduler:          simple, enabled, active
snap-id:      EaXqgt1lyCaxKaQCU349mlodBkDCXRcg
tracking:     stable
refresh-date: today at 13:44 CST
channels:
  stable:         v1.13.0  (340) 204MB classic
  candidate:      v1.13.0  (340) 204MB classic
  beta:           v1.13.0  (340) 204MB classic
  edge:           v1.13.1  (350) 229MB classic
  1.13/stable:    v1.13.0  (340) 204MB classic
  1.13/candidate: v1.13.0  (340) 204MB classic
  1.13/beta:      v1.13.0  (340) 204MB classic
  1.13/edge:      v1.13.1  (351) 229MB classic
  1.12/stable:    v1.12.3  (336) 226MB classic
  1.12/candidate: v1.12.3  (336) 226MB classic
  1.12/beta:      v1.12.3  (336) 226MB classic
  1.12/edge:      v1.12.3  (336) 226MB classic
  1.11/stable:    v1.11.5  (322) 219MB classic
  1.11/candidate: v1.11.5  (322) 219MB classic
  1.11/beta:      v1.11.5  (322) 219MB classic
  1.11/edge:      v1.11.5  (322) 219MB classic
  1.10/stable:    v1.10.11 (321) 175MB classic
  1.10/candidate: v1.10.11 (321) 175MB classic
  1.10/beta:      v1.10.11 (321) 175MB classic
  1.10/edge:      v1.10.11 (321) 175MB classic
installed:        v1.13.0  (340) 204MB classic
```

> 下指令操作環境請使用`User`執行，使用root user執行會`command not found`。

通過`microk8s.status`檢查 microk8s目前狀態：
```bash=
$ microk8s.status

microk8s is running
addons:
ingress: disabled
dns: disabled
metrics-server: disabled
istio: disabled
gpu: disabled
storage: disabled
dashboard: disabled
registry: disabled
```

## 訪問Kubernetes

為了避免與`kubectl`已安裝在原環境導致發生衝突並避免覆蓋任何現有的Kubernetes配置文件，MicroK8s添加了一個microk8s.kubectl命令，配置為專門訪問新的MicroK8s安裝。


```bash=
$ microk8s.kubectl get nodes

NAME          STATUS   ROLES    AGE     VERSION
k8s-testing   Ready    <none>   2m30s   v1.13.0
```

### microk8s.config

```bash=
$ microk8s.config

apiVersion: v1
clusters:
- cluster:
    server: http://192.0.0.10:8080
  name: microk8s-cluster
contexts:
- context:
    cluster: microk8s-cluster
    user: admin
  name: microk8s
current-context: microk8s
kind: Config
preferences: {}
users:
- name: admin
  user:
    username: admin
```

### microk8s.docker
```bash=
$ microk8s.docker -v
Docker version 17.03.2-ce, build f5ec1e2
```

### microk8s.start / stop
```bash=
$ microk8s.stop
Stopped.
$ microk8s.start
Started.
```

> 若無與原生`kubectl`衝突影響，這邊可以透過`alias`指定名稱省去輸入指令時間。

```bash=
$ sudo snap alias microk8s.kubectl kubectl
Added:
  - microk8s.kubectl as kubectl
```

> 解除alias設定

```bash=
$ snap unalias kubectl
```

## 啟動 microk8s.enable

默認情況下，我們在Kubernetes上游獲得了一個準系統。可以使用以下microk8s.enable命令運行其他服務，例如kube-dns和儀表板：

最重要的插件列表:

- dns：部署kube dns。其他人可能需要此插件，因此我們建議您始終啟用它。
- 儀表板：部署kubernetes儀表板以及grafana和Influxdb。
- storage：創建默認存儲類。此存儲類使用指向主機上目錄的hostpath-provisioner。
- ingress：創建入口控制器。
- gpu：通過啟用nvidia-docker運行時和nvidia-device-plugin-daemonset，將GPU暴露給microk8s。需要已在主機系統上安裝NVIDIA驅動程序。
- istio：部署核心Istio服務。您可以使用該microk8s.istioctl命令來管理部署。
- registry：部署docker私有註冊表並在localhost：32000上公開它。存儲插件將作為此插件的一部分啟用。

```bash=
$ microk8s.enable dashboard

Enabling dashboard
secret/kubernetes-dashboard-certs created
serviceaccount/kubernetes-dashboard created
deployment.apps/kubernetes-dashboard created
service/kubernetes-dashboard created
service/monitoring-grafana created
service/monitoring-influxdb created
service/heapster created
deployment.extensions/monitoring-influxdb-grafana-v4 created
serviceaccount/heapster created
configmap/heapster-config created
configmap/eventer-config created
deployment.extensions/heapster-v1.5.2 created
dashboard enabled
```

```bash=
$ microk8s.kubectl -n kube-system get pod,svc -o wide                                                                                           Sat Dec 15 14:48:45 2018

NAME                                                  READY   STATUS    RESTARTS   AGE     IP          NODE          NOMINATED NODE   READINESS GATES
pod/heapster-v1.5.2-64874f6bc6-nt72k                  4/4     Running   0          94s     10.1.1.17   k8s-testing   <none>           <none>
pod/kubernetes-dashboard-654cfb4879-xm8kp             1/1     Running   0          2m26s   10.1.1.13   k8s-testing   <none>           <none>
pod/monitoring-influxdb-grafana-v4-6679c46745-4kv5s   2/2     Running   0          2m25s   10.1.1.14   k8s-testing   <none>           <none>

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE     SELECTOR
service/heapster               ClusterIP   10.152.183.62    <none>        80/TCP              2m25s   k8s-app=heapster
service/kubernetes-dashboard   ClusterIP   10.152.183.54    <none>        443/TCP             2m26s   k8s-app=kubernetes-dashboard
service/monitoring-grafana     ClusterIP   10.152.183.188   <none>        80/TCP              2m26s   k8s-app=influxGrafana
service/monitoring-influxdb    ClusterIP   10.152.183.117   <none>        8083/TCP,8086/TCP   2m25s   k8s-app=influxGrafana
```

可以使用`disable`命令隨時停用這些插件

```bash=
$ microk8s.disable dashboard dns
```
可隨時透過`microk8s.status`你可以看到當前啟用與可用插件列表。


k8s barebone:
- api-server
- controller-manager
- scheduler
- kubelet
- cni
- kube-proxy

除了 Dashboard 和 private registry，還可以選擇`GPU`和`Istio`服務：

- DNS：部署kube dns。
- Dasnboard：部署kubernetes Dashboard 以及 grafana 和 Influxdb。
- storage：創建默認存儲區。此存儲使區指向主機上目錄的hostpath-provisioner。
- ingress
- gpu：通過啟用`nvidia-docker`運行時和`nvidia-device-plugin-daemonset`，將GPU掛給microk8s。需要已在主機系統上安裝`NVIDIA驅動程序`。
- istio：部署核心Istio服務。您可以使用該microk8s.istioctl命令來管理部署。
- registry

### 啟用GPU microk8s.enable gpu

```bash=
$ microk8s.enable gpu

Enabling NVIDIA GPU
NVIDIA kernel module detected
Enabling DNS
Applying manifest
service/kube-dns created
serviceaccount/kube-dns created
configmap/kube-dns created
deployment.extensions/kube-dns created
Restarting kubelet
DNS is enabled
Applying manifest
daemonset.extensions/nvidia-device-plugin-daemonset created
NVIDIA is enabled
```

檢查enable後`nvidia-divice-plugin`狀態：

```bash=
$ microk8s.status

microk8s is running
addons:
ingress: disabled
dns: enabled
metrics-server: disabled
istio: disabled
gpu: enabled
storage: disabled
dashboard: enabled
registry: disabled
```

```bash=
$ microk8s.kubectl -n kube-system get pod,svc -o wide

NAME                                                  READY   STATUS    RESTARTS   AGE   IP          NODE          NOMINATED NODE   READINESS GATES
pod/heapster-v1.5.2-64874f6bc6-wkxtm                  4/4     Running   4          14m   10.1.1.25   k8s-testing   <none>           <none>
pod/kube-dns-6ccd496668-gv9pm                         3/3     Running   0          38s   10.1.1.26   k8s-testing   <none>           <none>
pod/kubernetes-dashboard-654cfb4879-wbzkz             1/1     Running   1          14m   10.1.1.23   k8s-testing   <none>           <none>
pod/monitoring-influxdb-grafana-v4-6679c46745-n4j74   2/2     Running   3          14m   10.1.1.24   k8s-testing   <none>           <none>
pod/nvidia-device-plugin-daemonset-w828l              1/1     Running   0          32s   10.1.1.27   k8s-testing   <none>           <none>

NAME                           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE   SELECTOR
service/heapster               ClusterIP   10.152.183.11    <none>        80/TCP              14m   k8s-app=heapster
service/kube-dns               ClusterIP   10.152.183.10    <none>        53/UDP,53/TCP       38s   k8s-app=kube-dns
service/kubernetes-dashboard   NodePort    10.152.183.201   <none>        443:32560/TCP       14m   k8s-app=kubernetes-dashboard
service/monitoring-grafana     ClusterIP   10.152.183.191   <none>        80/TCP              14m   k8s-app=influxGrafana
service/monitoring-influxdb    ClusterIP   10.152.183.68    <none>        8083/TCP,8086/TCP   14m   k8s-app=influxGrafana
```

## 測試GPU節點是否可以正常運作

這邊簡易部署`gpu-pod`測試節點divice pligin 可以正常使用

```bash=
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  restartPolicy: Never
  containers:
  - image: nvidia/cuda
    name: cuda
    command: ["nvidia-smi"]
    resources:
      limits:
        nvidia.com/gpu: 1
EOF
pod "gpu-pod" created
```

```bash=
$ kubectl get pod -o wide                                                                                                                       Sat Dec 15 15:20:54 2018

NAME      READY   STATUS      RESTARTS   AGE     IP          NODE          NOMINATED NODE   READINESS GATES
gpu-pod   0/1     Completed   0          3m29s   10.1.1.28   k8s-testing   <none>           <none>
```

查看Pod log 狀態：
```bash=
$ kubectl logs gpu-pod
Sat Dec 15 07:18:53 2018
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 410.79       Driver Version: 410.79       CUDA Version: 10.0     |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 106...  Off  | 00000000:03:00.0 Off |                  N/A |
| 37%   25C    P8     5W / 120W |      0MiB /  6077MiB |      1%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|  No running processes found                                                 |
+-----------------------------------------------------------------------------+
```

## 刪除 microk8s

在移除microk8s之前，請先使用`microk8s.reset`停止所有正在運行的pod

```bash=
$ microk8s.reset
$ snap remove microk8s
```

## 相關參考
[[Github/microk8s]](https://github.com/ubuntu/microk8s)
[[Using GPGPUs with Kubernetes]](https://blog.ubuntu.com/2018/12/10/using-gpgpus-with-kubernetes?fbclid=IwAR05YY0s2aOifQQj6prD7phDyvdgXDJM_UIriMcwzsoLorni97ovVFnJ8yI)
[[install a local kubernetes with microk8s]](https://tutorials.ubuntu.com/tutorial/install-a-local-kubernetes-with-microk8s#0)
[[MicroK8sを使ってみる]](https://qiita.com/niiku-y/items/e5285af4f12b1318cf4e#snap-remove)
