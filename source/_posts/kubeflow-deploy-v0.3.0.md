---
title: 快速部署 kubeflow v0.3 容器機器學習平台
toc: true
date: 2018-10-23 20:53:45
categories: 
- Kubeflow
tags: 
- kubernetes
- kubeflow
- deploy
thumbnail: /images/kubeflow_deploy.jpeg
---

[Kubeflow](https://v0-3.kubeflow.org/) 是由 Google 與相關公司共同發起的開源專案，其目標是利用 Kubernetes 容器平台來簡化機器學習的建置與執行過程，使之更簡單、可重複的攜帶性與可擴展，並提供一套標準的雲原生(Cloud Native)機器學習解決方案，以幫助演算法科學家專注在模型開發本身，而由於 Kubeflow 以 Kubernetes 做基礎，因此只要有 Kubernetes 地方就能夠快速部署並執行。

<!-- more -->

- Kubeflow目標不是在於重建其他服務，而是提供一個最佳開發系統，來部署到任何集群中，有效確保ML在集群之間移動性，並輕鬆將任務擴展任何集群。

- 由於使用 Kubernetes 來做為基礎，因此只要有 Kubernetes 的地方，都能夠運行部署 Kubeflow。

> 在不同基礎設施上協助實現Machine learning 工作流程「更加簡單」與「便於攜帶」、「可擴展性」的部署，在不同基礎設施上(Local Laptop、GPU Server cluster、Cloud Production Cluster)



![](https://i.imgur.com/fPEG7eK.png)

## 簡述Kubelfow工作流程：

1. 下載`Kubeflow shell script` 和 配置文件
2. 自定義 `ksonnet `部署所需的JSON組件
3. 執行`shell script`將容器部署到所選擇的Kubernetes Enveroment
4. 開始執行機器學習訓練


我們透過Kubeflow工具管理，可以解決ML環境配置困難的問題，其中在Kubeflow提供Operator，對不同的機器學習框架，擴展的 Kubernetes API 進行自動建立、管理與配置應用程式容器實例，並提供訓練在不同環境和服務，包含數據準備、模型培訓、預測服務和服務管理。

> Kubernetes 有提供不同集群部署支援，包括單節點Minikube部署、 透過 Hypervisor 部署的 Multipass & Microk8s、Cloud。無論你在哪裡運行 Kubernetes，你都能夠運行 Kubeflow。


## 節點資訊

本次安裝作業系統採用Ubuntu 16.04 Destop，測試環境為實體機器部署 HA Cluster：

| IP Address	| Role	| vCPU	| RAM	| Extra Device| 
| -------- | -------- | -------- |-------- |-------- |
| 172.22.132.98	|VIP	|
| 172.22.132.81	| k8s-m1	| 8	| 16G	| GTX 1060 6G| 
| 172.22.132.82	| k8s-m2	| 8	| 16G	| GTX 1060 6G| 
| 172.22.132.83	| k8s-m3	| 8	| 16G	| GTX 1060 6G| 
| 172.22.132.84	| k8s-g1	| 8	| 16G	| GTX 1060 6G| 
| 172.22.132.97	| k8s-g2	| 8	| 16G	| GTX 1060 6G| 
| 172.22.132.86	| k8s-g3	| 8	| 16G	| GTX 1060 6G| 
| 172.22.132.94	| k8s-g11	| 8	| 16G	| GTX 1060 6G| 


> kubernetes部署可以參考 [開發 Ansible Playbooks 部署 Kubernetes v1.11.x HA 叢集](https://kairen.github.io/2018/08/12/kubernetes/deploy/k8s-ansible-ha/)教學

###  本次部署使用版本
> - kubernetes v1.11.2
>     - Master x 3, Worker x 4
> - ksonnet 0.13.0 
> - kubeflow v0.3.0 

## 事前準備

使用 Kubeflow 之前，需要確保以下條件達成：

- 所有節點確認已經安裝 [`NVIDIA driver`](https://www.nvidia.com.tw/content/DriverDownload-March2009/confirmation.php?url=/Mac/cuda_396/cudadriver_396.64_macos.dmg&lang=tw&type=GeForce)、[`CUDA`](https://developer.nvidia.com/cuda-downloads)、[`Docker`](https://docs.docker.com/install/linux/docker-ce/ubuntu/)、[`NVIDIA Docker`](https://github.com/NVIDIA/nvidia-docker)，以下為實驗環境版本資訊：

    - CUDA Version: 10.0.130
    - Driver Version: 410.48
    - Docker Version: 18.03.0-ce
    - NVIDIA Docker: 2.0.3

### 安裝 NFS 

設置Kubeflow需要Volumes來存儲數據，建立 NFS server 並在 Kubernetes 節點安裝 NFS common，然後利用 Kubernetes 建立 PV 提供給 Kubeflow 使用：

```bash=
# 在 master 執行，這邊nfs-server建立在k8s-m1節點
$ sudo apt-get update && sudo apt-get install -y nfs-server
$ sudo mkdir /nfs-data
# 可以在 `nfs-data`資料夾建立多個pv資料夾，提供kubeflow儲存使用
$ cd /nfs-data
$ mkdir user1 user2 ... 
$ echo "/nfs-data *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports
$ sudo /etc/init.d/nfs-kernel-server restart

# 在 node 執行
$ sudo apt-get update && sudo apt-get install -y nfs-common
```

> 這邊也可以使用其他存儲數據方式: [Kubernetes: Local Volume](https://qiita.com/ysakashita/items/8b411943f7e25dc4c5b0)

### 安裝 Ksonnet 

Ksonnet 是一個命令工具，可以更輕鬆地管理由多個組件組成的複雜部署，簡化編寫和部署 Kubernetes 配置。


```bash=
$ wget https://github.com/ksonnet/ksonnet/releases/download/v0.13.0/ks_0.13.0_linux_amd64.tar.gz
$ tar xvf ks_0.13.0_linux_amd64.tar.gz
$ sudo cp ks_0.13.0_linux_amd64 /usr/local/bin/
$ chmod +x ks

$ ks version
ksonnet version: 0.13.0
jsonnet version: v0.11.2
client-go version: kubernetes-1.10.4
```

## 部署 Kubeflow

本節將說明如何利用 ksonnet 來部署 Kubeflow 到 Kubernetes 叢集中。部署版本選擇為[kubeflow v0.3.0](https://v0-3.kubeflow.org/)。

### 下載 Kubeflow 腳本

下載GitHub中Kubeflow設置所需配置：

```bash=
# 建立配置指定資料夾路徑，可自行更換名稱
$ KUBEFLOW_SRC=mykfsrc   
$ mkdir ${KUBEFLOW_SRC}
$ cd ${KUBEFLOW_SRC}

# 部署kubeflow版本標籤，可自行選擇其他分支
$ export KUBEFLOW_TAG=v0.3.0 

# 執行shell script
$ curl https://raw.githubusercontent.com/kubeflow/kubeflow/${KUBEFLOW_TAG}/scripts/download.sh | bash 
```

下載後目錄下會多`kubeflow`、`scripts`資料夾:

```bash=
$ tree -L 1                                               
├── kubeflow
└── scripts
```

### 執行腳本配置和部署Kubeflow
```bash=
# 退回上一層建立對應路徑
$ cd ../
$ KUBEFLOW_REPO=$(pwd)/mykfsrc

# ${KFAPP} 為配置 kubeflow 儲存目錄的名稱，執行init時將創建此目錄
# ksonnet應用程式將在 ${KFAPP}/ks_app目錄中創建
$ KFAPP=mykfapp
$ ${KUBEFLOW_REPO}/scripts/kfctl.sh init ${KFAPP} --platform none

# 安裝 Kubeflow 套件至應用程式目錄
$ cd ${KFAPP}
$ ${KUBEFLOW_REPO}/scripts/kfctl.sh generate k8s

# 定義 Namespace 為kubeflow來管理相關資源
$ kubectl create ns kubeflow

# 部署 Kubeflow
$ ${KUBEFLOW_REPO}/scripts/kfctl.sh apply k8s
```

> ！部署預設為default，這邊部署有特別給kubeflow 命名空間，如果要移除kubeflow，直接刪除kubeflow Namespaces 時，所有物件也會被刪除

## 完成後檢查 Kubeflow 元件部署結果：

```bash=
$ kubectl -n kubeflow get po -o wide                
ambassador-868d5dbdc4-5fgdw                 3/3       Running            0          7m        10.244.2.90    k8s-g3    <none>
ambassador-868d5dbdc4-qcp5z                 3/3       Running            1          7m        10.244.5.78    k8s-g11   <none>
ambassador-868d5dbdc4-x9bfc                 3/3       Running            0          7m        10.244.3.96    k8s-g2    <none>
argo-ui-84464bd59c-hmhmd                    1/1       Running            0          7m        10.244.4.89    k8s-g1    <none>
centraldashboard-c76877875-nqbhg            1/1       Running            0          7m        10.244.3.97    k8s-g2    <none>
modeldb-backend-58969447f6-z2xmf            1/1       Running            1          11s       10.244.3.102   k8s-g2    <none>
modeldb-db-57b855f5b7-dw5s6                 1/1       Running            0          7m        10.244.5.79    k8s-g11   <none>
modeldb-frontend-769d5bdd66-njkqs           1/1       Running            0          7m        10.244.3.99    k8s-g2    <none>
spartakus-volunteer-5bfd5876fb-b86h5        1/1       Running            0          7m        10.244.2.91    k8s-g3    <none>
studyjob-controller-56588dc6f9-shg72        1/1       Running            0          7m        10.244.3.101   k8s-g2    <none>
tf-hub-0                                    1/1       Running            0          7m        10.244.4.86    k8s-g1    <none>
tf-job-dashboard-7777b6bf-qltfc             1/1       Running            0          7m        10.244.3.98    k8s-g2    <none>
tf-job-operator-v1alpha2-5c5b4dcfdf-x5rkv   1/1       Running            0          7m        10.244.4.87    k8s-g1    <none>
vizier-core-5584ccbd8-l2bxr                 0/1       CrashLoopBackOff   4          7m        10.244.5.80    k8s-g11   <none>
vizier-db-547967c899-mf988                  0/1       Pending            0          7m        <none>         <none>    <none>
vizier-suggestion-grid-8547dbb55b-k8tw6     1/1       Running            0          7m        10.244.4.90    k8s-g1    <none>
vizier-suggestion-random-d7c5cd68b-7p9nz    1/1       Running            0          7m        10.244.3.100   k8s-g2    <none>
workflow-controller-5c95f95f58-d9plp        1/1       Running            0          7m        10.244.4.88    k8s-g1    <none>
```

可以看到`vizier`狀態為Pending，失敗的原因是PersistentVolume未分配給`vizer-db`（實際是MySQL）。所以`vizier-db`正在等待並且未能啟動vizier-core。

這邊透過使用`kubectl describe`命令檢查原因:

```bash=
$ kubectl -n kubeflow describe pod vizier-db-547967c899-mf988
Events:
  Type     Reason            Age                 From               Message
  ----     ------            ----                ----               -------
  Warning  FailedScheduling  2m (x417 over 12m)  default-scheduler  pod has unbound PersistentVolumeClaims (repeated 4 times)

$ kubectl -n kubeflow get pvc 
NAME        STATUS    VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
vizier-db   Pending                                                      15m
```

這邊我們建立一個 NFS PV 來提供給 `vizier-db` 使用：
```bash=
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: vizier-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.22.132.81
    path: /nfs-data/kubeflow-pv1
EOF
```
> 請更換自己的 nfs-server IP


再次查看 PV & PVC 狀態，就可以看到`vizier-db`獲得 20Gi 的  ersistentVolume 空間

```bash=
$ kubectl get pv                                         
NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS    CLAIM                STORAGECLASS   REASON    AGE
vizier-pv   20Gi       RWO            Retain           Bound     kubeflow/vizier-db                            16s

$ kubectl -n kubeflow get pvc                           
NAME        STATUS    VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS   AGE
vizier-db   Bound     vizier-pv   20Gi       RWO                           20m
```

> 其他儲存方式參考：[Kubernetes: Local Volumeの検証](https://qiita.com/ysakashita/items/67a452e76260b1211920)

> 暫時問題：modeldb-backend 會不定時 Crash
---

## Kubeflow UI 
查看Service中，可以看到kubeflow v0.3.0版本中，附帶了許多Web UI:
- Argo UI
- Central UI for navigation
- JupyterHub
- Katib
- TFJobs Dashboard

這邊我們透過Service查看一下目前pod狀態：

```bash=
$ kubectl -n kubeflow get svc  -o wide                   
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE       SELECTOR
ambassador                 ClusterIP   10.98.216.151    <none>        80/TCP           7m        service=ambassador
ambassador-admin           ClusterIP   10.102.20.172    <none>        8877/TCP         6m        service=ambassador
argo-ui                    NodePort    10.102.36.121    <none>        80:32530/TCP     6m        app=argo-ui
centraldashboard           ClusterIP   10.99.255.3      <none>        80/TCP           6m        app=centraldashboard
k8s-dashboard              ClusterIP   10.110.209.78    <none>        443/TCP          6m        k8s-app=kubernetes-dashboard
modeldb-backend            ClusterIP   10.102.179.187   <none>        6543/TCP         6m        app=modeldb,component=backend
modeldb-db                 ClusterIP   10.102.143.161   <none>        27017/TCP        6m        app=modeldb,component=db
modeldb-frontend           ClusterIP   10.100.76.92     <none>        3000/TCP         6m        app=modeldb,component=frontend
statsd-sink                ClusterIP   10.98.167.5      <none>        9102/TCP         7m        service=ambassador
tf-hub-0                   ClusterIP   None             <none>        8000/TCP         6m        app=tf-hub
tf-hub-lb                  ClusterIP   10.106.99.192    <none>        80/TCP           6m        app=tf-hub
tf-job-dashboard           ClusterIP   10.100.66.145    <none>        80/TCP           6m        name=tf-job-dashboard
vizier-core                NodePort    10.105.154.126   <none>        6789:30678/TCP   6m        app=vizier,component=core
vizier-db                  ClusterIP   10.100.32.162    <none>        3306/TCP         6m        app=vizier,component=db
vizier-suggestion-grid     ClusterIP   10.105.68.247    <none>        6789/TCP         6m        app=vizier,component=suggestion-grid
vizier-suggestion-random   ClusterIP   10.110.164.76    <none>        6789/TCP         6m        app=vizier,component=suggestion-random
```

這時候就可以透過 kubeflow UI，能引導到 登入JupyterHub，但這邊需要修改 Kubernetes Service，透過以下指令進行：

```bash=
# 修改 svc 將 Type 修改成 LoadBalancer，並且新增 externalIPs 指定為 Master IP。
$ kubectl -n kubeflow edit svc ambassador
...
sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
...

# 修改後就可以透過 NodePort 接上 Port 進入顯示 Jupyter Notebook
NAME                       TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE       SELECTOR
ambassador                 NodePort    10.98.216.151    <none>        80:32752/TCP     15m       service=ambassador
```

完成後連接 `http://Master_IP:Port`即可看到kubeflow UI。

![](https://i.imgur.com/dH1fh00.png)

可以透過上方連結查看`Jupyter Hub`、`TFjob Dashboard`、`Kubernetes Dashboard` 等。

![](https://i.imgur.com/pBzrG66.png)


## 測試 Jupyter Notebook


這邊需要再次先建立一個 NFS PV 來提供給 `Kubeflow Jupyter`使用：

```bash=
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: notebook-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: 172.22.132.81
    path: /nfs-data/kubeflow-pv2
EOF
```

NFS PV 建立好後這邊UI點選`Jupyter Notebook`登入測試，並輸入`任意帳號密碼`進行登入：

![](https://i.imgur.com/1wwlLqP.png)
Spawner Optinos 這邊可以輸入Notebook Pod節點資源，包含Image、CPU、Memory、GPU數量，最後點選`Spawn`來完成建立 Server，如下圖所示：

![](https://i.imgur.com/9pyuZqp.png)


kubeflow v0.3版本開始加入，在部署Jupyter Hub時的Event log方便了解部署狀態
![](https://i.imgur.com/BmpbGhB.png)

部署完後即可直接操作擁有GPU的Jupyter Notebook
![](https://i.imgur.com/Udsm6ng.png)


### 補充: 透過TFJobs Dashboard 顯示`TFJob`狀態

可以透過[TensorFlow Training (TFJob)](https://www.kubeflow.org/docs/guides/components/tftraining/)定義TFjob，即可從`TFJobs Dashboard`查看Job資訊，也可以直接透過Dashboard建立/刪除TFJob。

> 這邊透單測試Tensorflow Benchmark GPU測試 [範例](https://github.com/yylin1/tf-operator-benchmarks/blob/master/tf-cnn/tf-benchmark-gpu-example.yaml)

![](https://i.imgur.com/FreVFsv.png)

查看 `PS`狀態:
![](https://i.imgur.com/txqyKGf.png)


查看 `Worker`狀態:
![](https://i.imgur.com/GsMPOwy.png)


## 查看GPU資源狀態
可以透過執行以下指令確認是否GPU可被分配資源：
```bash=
$ kubectl get nodes "-o=custom-columns=NAME:.metadata.name,GPU:.status.allocatable.nvidia\.com/gpu"
NAME      GPU
k8s-g1    1
k8s-g11   1
k8s-g2    1
k8s-g3    1
k8s-m1    <none>
k8s-m2    <none>
k8s-m3    <none>
```

後續有陸續關注，關於 kubeflow CLI 的專案[`kubeflow/Arena`](https://github.com/kubeflow/arena)，主要供資料科學家運行和監控機器學習培訓工作並以簡單的方式檢查其結果。目前它支持單獨/分佈式TensorFlow訓練。部署基於Kubernetes，helm和Kubeflow。

同時，最終用戶需要 GPU 資源和節點管理。Arena還提供top檢查Kubernetes集群中可用GPU資源的命令。

```bash=
$ arena top node                                     
NAME     IPADDRESS      ROLE    GPU(Total)  GPU(Allocated)
k8s-g1   172.22.132.84  <none>  1           0
k8s-g11  172.22.132.94  <none>  1           0
k8s-g2   172.22.132.97  <none>  1           0
k8s-g3   172.22.132.86  <none>  1           1
k8s-m1   172.22.132.81  master  0           0
k8s-m2   172.22.132.82  master  0           0
k8s-m3   172.22.132.83  master  0           0
-----------------------------------------------------------------------------------------
Allocated/Total GPUs In Cluster:
1/5 (20%)
```

> 透過CLI方式來 `watch -n1 arena top node`就能很清楚知道節點資源使用狀況。

## 移除 kubeflow

這邊部署有特別給kubeflow 命名空間，如果要移除kubeflow，直接刪除kubeflow Namespaces 時，所有物件也會被刪除

```bash=
kubectl -n kubeflow delete namespace kubeflow
```

相關文章:
[kubeflow] Local Volumeを使ったKubeflowの動作検証
[kubeflow] Microk8s for Kubeflow

**#! 更多相關kubeflow Components實作會在後續文章陸續介紹**
