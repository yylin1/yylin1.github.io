---
title: 利用 Minikube 快速測試部署 Kubeflow v0.5
toc: true
date: 2019-05-03 23:46:54
categories: 
- Kubeflow
tags: 
- minikube
- kubeflow
- deploy
- quickstart
thumbnail: /images/minikube_for_kubeflow.jpg
---

- 本文介紹如何在 Minikube 上運行 Kubeflow v0.5 基本安裝步驟
- Minikube 會透過虛擬機（VM）運行簡單的單節點 Kubernetes 叢集
- 文章最後，你將擁有 Minikube 單節點kubernetes 叢集安裝以及Kubeflow的所有默認核心 Components
- 最後能夠訪問 kubeflow UIs 包含 `Jupyter Notebook`、`katib`、`pipeline`

<!-- more -->

此篇文章介紹「實驗環境」採用 [@Kyle Bai](https://github.com/kairen)  `修正過後的 minikube` 進行測試，如果環境資源充足想測試多節點 `minikube`，詳細可以透過參考文章步驟，快速建立多節點的 minikube 叢集：「[利用 Minikube 快速建立測試用 Kubernetes 叢集](https://k2r2bai.com/2019/01/22/kubernetes/deploy/minikube-multi-node/)」


- **R-Ladies Taipei x Cloud Native Taiwan User Group Meetup 分享**
    - 投影片：[Kubeflow 對於機器學習平台的願景](https://speakerdeck.com/yylin1/kubeflow-dui-yu-ji-qi-xue-xi-ping-tai-de-yuan-jing)

---

## 配置 Minikube 環境

**首先請事先下載 Minikube、Virtualbox 與 kubectl 於當前實驗環境：**
- [Virtual Box](https://www.virtualbox.org/wiki/Downloads)
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)

**Minikube （請依據自己當下實驗環境選擇下載）**
- [Linux](https://github.com/kairen/minikube/releases/download/v0.33.1-multi-node/minikube-linux-amd64)
- [Mac OS X](https://github.com/kairen/minikube/releases/download/v0.33.1-multi-node/minikube-darwin-amd64)
- [Windows](https://github.com/kairen/minikube/releases/download/v0.33.1-multi-node/minikube-windows-amd64.exe)

> - **IMPORTANT**: 測試機器記得開啟 VT-x or AMD-v virtualization.
> - Windows 使用者建議安裝 [Git bash](https://gitbook.tw/chapters/environment/install-git-in-windows.html) or [Cmder](https://cmder.net/) 來操作。

---
### 啟動 Minikube

如果自己機器已經有 Minikube 的話，就把上面的 binary 檔放任意方便你執行的位置，然後再開始前請先刪除 Home 目錄的`.minikube`資料夾：

```shell=
$ rm -rf $HOME/.minikube
```

當刪除上面檔案後，即可透過 Minikube 啟動單節點叢集:

```shell=
$ minikube start --memory=8192 --cpus=4 --disk-size=40g

// 完成啟動部署後顯示
Everything looks great. Please enjoy minikube!
```
> - `--vm-driver` 可以選擇使用其他 VM driver 來啟動虛擬機，如 xhyve、hyperv、hyperkit 與 kvm2 等等。

```shell=
$ kubectl -n kube-system get po
NAME                               READY     STATUS    RESTARTS   AGE
calico-node-tcdmg                  2/2       Running   0          92s
coredns-86c58d9df4-27ktl           1/1       Running   0          96s
coredns-86c58d9df4-6gj4x           1/1       Running   0          96s
etcd-minikube                      1/1       Running   0          45s
kube-addon-manager-minikube        1/1       Running   0          114s
kube-apiserver-minikube            1/1       Running   0          67s
kube-controller-manager-minikube   1/1       Running   0          74s
kube-proxy-pthlp                   1/1       Running   0          96s
kube-scheduler-minikube            1/1       Running   0          60s
storage-provisioner                1/1       Running   0          83s
```

確認節點處於 Ready 狀態：
```shell=
$ kubectl get no
NAME       STATUS    ROLES     AGE       VERSION
minikube   Ready     master    2m33s     v1.13.2

$ kubectl get cs
NAME                 STATUS    MESSAGE              ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health": "true"}
```
---
### 安裝 Ksonnet

[Ksonnet](https://ksonnet.io/) 是一個命令工具，組件配置參數按照環境 `env` 來組成，可以更輕鬆地管理由多個複雜部署到相應的 Cluster，簡化編寫和部署 component。

- Linux 

```shell=
$ wget https://github.com/ksonnet/ksonnet/releases/download/v0.13.0/ks_0.13.0_linux_amd64.tar.gz
$ tar xvf ks_0.13.0_linux_amd64.tar.gz
$ sudo cp ks_0.13.0_linux_amd64 /usr/local/bin/
$ chmod +x ks

$ ks version
ksonnet version: 0.13.0
jsonnet version: v0.11.2
client-go version: kubernetes-1.10.4
```

- Mac OS X

```shell=
$ brew install ksonnet/tap/ks
```


> 補充官網: [kubeflow 安裝 ksonnet 說明](https://www.kubeflow.org/docs/components/ksonnet/)
---

### 透過 kfctl 部署 kubeflow v0.5

在 kubeflow v0.5 版本不在是透過 script `kfctl.sh` 腳本執行，改成透過 golang 所生成的 binary file `kfctl` 來進行kubeflow部署。

- 下載 kfctl release 版本 [Kubeflow releases page](https://github.com/kubeflow/kubeflow/releases/).

![](https://i.imgur.com/rb6eCGI.png)

> 請依照自己當前環境下載 kfctl binary 

```shell=
kfctl_<release tag>_<platform>.tar.gz
```

- Mac OS X 

```shell=
$ wget kfctl_v0.5.0_darwin.tar.gz
$ tar -xvf kfctl_v0.5.0_darwin.tar.gz
x ./kfctl
```

- 配置 `kfctl` binary 執行路徑
- 定義 {KFAPP} 資料夾目錄名稱

```shell=
$ export PATH=$PATH:<path to kfctl in your kubeflow installation>
# Default uses IAP. 可自行定義
$ export KFAPP=kubeflow
```

```shell=
#1 
$ kfctl init ${KFAPP}
#2 
$ cd ${KFAPP}
$ kfctl generate all -V
#3 
$ kfctl apply all -V
```

> 補充：[kubeflow 官網詳細 deploy 資訊](https://www.kubeflow.org/docs/started/getting-started-k8s/#deploy-kubeflow)

---

### #1 初始化 kubeflow components
執行 `kfctl init ${KFAPP}` 生成 **app.yaml** 定義中包含 components、package 配置

```shell=
$ kfctl init ${KFAPP}
$ cat kubeflow/app.yaml

apiVersion: kfdef.apps.kubeflow.org/v1alpha1
kind: KfDef
metadata:
  creationTimestamp: null
  name: kubeflow
  namespace: kubeflow
spec:
  appdir: /Users/yylin/kubeflow/v0.5.0/minikube/kubeflow
  componentParams:
    ambassador:
    - name: ambassadorServiceType
      value: NodePort
  components:
  - metacontroller
  - ambassador
  - argo
  - centraldashboard
  - jupyter-web-app
  - katib
  - notebook-controller
  - pipeline
  - pytorch-operator
  - tensorboard
  - tf-job-operator
  packages:
  - argo
  - common
  - examples
  - gcp
  - jupyter
  - katib
  - metacontroller
  - modeldb
  - mpi-job
  - pipeline
  - pytorch-job
  - seldon
  - tensorboard
  - tf-serving
  - tf-training
  repo: /Users/yylin/kubeflow/v0.5.0/minikube/kubeflow/.cache/master/kubeflow
  useBasicAuth: false
  useIstio: false
  version: master
status: {}
```

---

### #2 生成config

```shell=
$ cd ${KFAPP}
$ kfctl generate all -V
```

創建 `config files` 定義各種資源，components 部署相關資訊會透過`.jsonnet`來執行

```
$ tree -L 3
.
├── app.yaml
└── ks_app
    ├── app.yaml
    ├── components
    │   ├── ambassador.jsonnet
    │   ├── argo.jsonnet
    │   ├── centraldashboard.jsonnet
    │   ├── jupyter-web-app.jsonnet
    │   ├── katib.jsonnet
    │   ├── metacontroller.jsonnet
    │   ├── notebook-controller.jsonnet
    │   ├── params.libsonnet
    │   ├── pipeline.jsonnet
    │   ├── pytorch-operator.jsonnet
    │   ├── tensorboard.jsonnet
    │   └── tf-job-operator.jsonnet
    ├── environments
    │   ├── base.libsonnet
    │   └── default
    ├── lib
    │   └── ksonnet-lib
    └── vendor
        └── kubeflow
        
8 directories, 15 files
```

---

### #3 部署

最後等待部署完成，這部分會花比較長的時間，過程會需要安裝 `Components`所需要的Container (依照環境網路，所有Images下載部署完成時間約20~30分鐘)

```shell=
$ kfctl apply all -V
...
...
INFO[0190] All components apply succeeded                filename="ksonnet/ksonnet.go:192"
```

- 完成 kubeflow 部署後會顯示 `All components apply succeeded `
- 確認環境狀態

```shell=
$ kubectl -n kubeflow get pod
NAME                                                       READY     STATUS    RESTARTS   AGE
ambassador-b4d9cdb8-mng2c                                  1/1       Running   0          42m
ambassador-b4d9cdb8-srj7k                                  1/1       Running   0          42m
ambassador-b4d9cdb8-zlxpv                                  1/1       Running   0          42m
argo-ui-cf5f88f78-d5h5d                                    1/1       Running   0          41m
centraldashboard-c967f47b4-5xc8c                           1/1       Running   0          41m
jupyter-web-app-7cc9c874bb-gzkfb                           1/1       Running   0          41m
katib-ui-7c6887b55f-rbbbc                                  1/1       Running   0          41m
metacontroller-0                                           1/1       Running   0          42m
minio-8ccf48d9d-h69r6                                      1/1       Running   0          40m
ml-pipeline-7885bc55dd-txmmz                               1/1       Running   0          40m
ml-pipeline-persistenceagent-5564f6f75f-vpmhm              1/1       Running   1          40m
ml-pipeline-scheduledworkflow-785bc5455c-l9mqn             1/1       Running   0          40m
ml-pipeline-ui-66bf476679-xm4pm                            1/1       Running   0          40m
ml-pipeline-viewer-controller-deployment-c84779fc5-p9ncm   1/1       Running   0          40m
mysql-ffc889689-mldkf                                      1/1       Running   0          40m
notebooks-controller-767b89b4d8-z5qj5                      1/1       Running   0          40m
pytorch-operator-6cd9dbf586-7zwhp                          1/1       Running   0          39m
studyjob-controller-855c6d8f84-f88fr                       1/1       Running   0          41m
tensorboard-5b57bff466-rdjkc                               1/1       Running   0          39m
tf-job-dashboard-6bddf98bc8-zzw7z                          1/1       Running   0          39m
tf-job-operator-68455999cc-2w7vv                           1/1       Running   0          39m
vizier-core-ccc5d9f5c-hpww6                                1/1       Running   0          41m
vizier-core-rest-555957f4dc-tt59h                          1/1       Running   0          41m
vizier-db-86dc7d89c5-fxcwk                                 1/1       Running   0          41m
vizier-suggestion-bayesianoptimization-775474d6c9-xg4m8    1/1       Running   0          41m
vizier-suggestion-grid-85d49b956f-ttqq2                    1/1       Running   0          41m
vizier-suggestion-hyperband-54567d484f-v6jrb               1/1       Running   0          41m
vizier-suggestion-random-64b5867867-wctrv                  1/1       Running   0          41m
workflow-controller-cf79dfbff-j5n8x                        1/1       Running   0          41m
```


---

## 總整理-理解部署過程 
部署過程由四個 `CLI` 命令執行完成:

- **init** - 初始化 kubeflow 安裝相關設定.
    - 生成 **app.yaml** 定義中包含 components、package 配置
     
- **generate** - 創建 `config files` 定義各種資源
- **apply** - 創建和更新部署資源
- **delete** - 刪除部署資源


kfctl 除`init` 之外，所有命令都可以使用參數來指定要執行叢集資源集：
- k8s - all resources that run on Kubernetes.
- all - GCP and Kubernetes resources.


---
## 進入 kubeflow UIs


> 補充：[kubeflow UI 介紹說明](https://www.kubeflow.org/docs/other-guides/accessing-uis/)

```shell=
$ export NAMESPACE=kubeflow
kubectl port-forward svc/ambassador -n ${NAMESPACE} 8080:80

$ kubectl port-forward svc/ambassador -n ${NAMESPACE} 8080:80
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
```

透過本地端連線即可看到 `kubeflow UI`
```
http://localhost:8080/
```

![](https://i.imgur.com/L1luU39.png)

--- 

## 刪除 Kubeflow

如果測試完畢需要移除`kubeflow`

```shell=
$ cd ${KFAPP}
# If you want to delete all the resources, including storage.
$ kfctl delete all --delete_storage
# If you want to preserve storage, which contains metadata and information
# from mlpipeline.
$ kfctl delete all
```

----

## 暫時退出 minikube 

```shell=
$ minikube stop
Stopping local Kubernetes cluster...
Machine stopped.

$ minikube status
host: Stopped
kubelet:
apiserver:
kubectl:
```

## 刪除 minikube 虛擬機與檔案

```shell=
$ minikube delete
$ rm -rf $HOME/.minikube
```
- 而檔案只要刪除 Home 目錄的 `.minikube` 資料夾，以及 `minikube` 執行檔即可


