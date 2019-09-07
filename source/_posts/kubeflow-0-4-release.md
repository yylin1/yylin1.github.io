---
title: kubeflow v0.4.x 部署與概述
toc: true
date: 2019-02-03 13:28:29
categories: 
- Kubeflow
tags: 
- kubernetes
- kubeflow
- deploy
thumbnail: /images/kubeflow-deploy-0-4.png
---

**Kubeflow 專案致力於使 ML Workflow 發展更具簡易性、可攜帶性和規模性。** [Kubeflow於2017年底正式開源，並於2018年5月發布了首個0.1版本](https://www.itcodemonkey.com/article/12711.html)，至今 kubeflow 各項目發展迅速，而我參與 `2018 KubeCon Shanghai` 聆聽好幾場來自 各大企業與 Google等議程，分享 Kubeflow 導入企業實際應用情境，深刻感受到 **kubeflow** 在開源社區與貢獻者中重視度，在快速發展下至於1月宣布 Kubeflow 0.4 穩定版本正式發佈。

#此篇主要記錄操作 kubflow v0.4.1 部署與更新概述。

### Related Posts
- [[kubeflow] 快速部署 Kubeflow v0.3 容器機器學習平台](https://yylin1.github.io/2018/10/23/kubeflow-deploy-v0.3.0/)
<!-- more -->

**# Quickly get running with your ML Workflow**

![](https://i.imgur.com/oU86YTx.png)

- Kubeflow目標不是在於重建其他服務，而是提供一個最佳開發系統（更直接的）部署到任何基礎設施中，有效確保 ML 在之間移動性，並輕鬆將任務擴展至任何基礎設施。

- 由於透過 Kubernetes 來做為底層基礎，因此只要有 Kubernetes 的地方，都能夠運行快速部署 Kubeflow。

使用Kubeflow的新型Jupyter筆記本電腦產品，創建筆記本電腦比以往更容易。選擇已安裝最流行的機器學習庫的精選圖像，或帶上您自己的自定義圖像。

---
## Kubeflow 0.4提供了以下主要功能和更新：

- 更新 [JupyterHub UI](https://github.com/kubeflow/kubeflow/tree/master/components/jupyterhub) 介面，可以更輕鬆生成附有`PVC`的 Jupyter Notebook，直觀選擇常用或自定義 Image，提供方便管理創建對 CPU / Memory / GPU 資源具有更多配置並生成 Jupyter notebooks 
-  alpha版本 [fairing](https://github.com/kubeflow/fairing)，一個簡化Library數據科學家建立並直接從 Notebook 開始訓練任務模型
- 用於 ML 工作流的 [Kubeflow Pipelines](https://github.com/kubeflow/pipelines)，通過重啟具有不同數據資料與更新數據pipeline來加速 productizing models 過程
- [Katib](https://github.com/kubeflow/katib) 支援 TFJob，使更容易比較不同的超參數性能來進行模型優化
- TFJob 和 PyTorch operators 升為 **Beta**版本，使數據科學家能夠針對更穩定 API對其訓練任務進行編譯，並更容易切換在訓練之間使用的框架

### Reference

- [Kubeflow 0.4 Release: Enhancements for Machine Learning productivity
](https://medium.com/kubeflow/kubeflow-0-4-release-enhancements-for-machine-learning-productivity-d77c54df07a9)

--- 

## 事前準備

#### 部署 Kubeflow 之前，需要確保以下條件達成：

所有 kubernetes 叢集`節點`確認已經安裝 NVIDIA Driver、CUDA、Docker、NVIDIA Docker，以下此篇文章部署實驗環境版本資訊 (kubernetes v1.11.2)：

- CUDA Version: 10.0.130
- Driver Version: 410.48
- Docker Version: 18.03.0-ce
- NVIDIA Docker: 2.0.3

>  Storage 儲數據方式是採用建立 `NFS server`，配置 PV 提供給 Kubeflow 使用：
> - 這邊參考先前文章設置: [安裝 NFS](https://yylin1.github.io/2018/10/23/kubeflow-deploy-v0.3.0/)
> - 其他存儲數據方式: [Kubernetes: Local Volume](https://qiita.com/ysakashita/items/8b411943f7e25dc4c5b0)

## 部署 Kubeflow

### 安裝ksonnet

Kubeflow 利用 [ksonnet](https://ksonnet.io/) 打包和部署其組件配置，可以更輕鬆地管理複雜部署。

安裝 [ksonnet v0.3.1](https://www.kubeflow.org/docs/components/ksonnet/) 版本 

```bash=
# 確保環境具有 ksonnet 最低要求版本
$ export KS_VER=0.13.1
$ export KS_PKG=ks_${KS_VER}_linux_amd64

$ wget ${KS_PKG}.tar.gz https://github.com/ksonnet/ksonnet/releases/download/v${KS_VER}/${KS_PKG}.tar.gz \
  --no-check-certificate
$ tar xvf $KS_PKG.tar.gz
$ sudo cp $KS_PKG/ks /usr/local/bin/
$ chmod +x /usr/local/bin/ks
```

確認 ksonnet 版本

```bash=
$ ks version
ksonnet version: 0.13.1
jsonnet version: v0.11.2
client-go version: kubernetes-1.10.4
```

### 部署儲存需求 PVC 

在kubeflow v0.4版本中，Kubeflow 依賴 `katib-mysql`、`pipeline-mino`、`pipeline-mysql`這三個有狀態服務。而這些服務需要提前部署儲存空間：

> 實驗為測試採用快速部署，所以簡單建立三個 `NFS PV` 儲存空間供給服務使用:
> - 請更換自己環境中 `{NFS-server IP}` 

```bash=
$ for i in $(seq 1 3); do
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
    name: kubeflow-pv${i}
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: {NFS-server IP}
    path: /nfs-data/kubeflow-pv
EOF
 done

...
persistentvolume/kubeflow-pv1 created
persistentvolume/kubeflow-pv2 created
persistentvolume/kubeflow-pv3 created
```

### 下載 Kubeflow 相關 repo

本節將說明如何利用 ksonnet 來部署 Kubeflow 到 Kubernetes 叢集中

下載GitHub中Kubeflow設置所需配置：

```bash=
# 建立配置指定資料夾路徑，可自行更換名稱
$ KUBEFLOW_SRC=mykfsrc   
$ mkdir ${KUBEFLOW_SRC}
$ cd ${KUBEFLOW_SRC}

# 部署kubeflow版本標籤，可自行選擇其他分支
$ export KUBEFLOW_TAG=v0.4.1 

# 執行shell script (複製時注意`\`)
$ curl https://raw.githubusercontent.com/kubeflow/kubeflow/${KUBEFLOW_TAG}/scripts/download.sh | bash
```

下載後 `$KUBEFLOW_SRC` 目錄下會多kubeflow、scripts資料夾:

```bash=
$ tree -L 1
.
├── deployment
├── kubeflow
└── scripts
```

## 執行安裝腳本配置與部署

```bash=
# 退回上一層建立對應路徑，配置 $KUBEFLOW_REPO
$ cd ../
$ KUBEFLOW_REPO=$(pwd)/mykfsrc

# ${KFAPP} 為配置 kubeflow 儲存目錄的名稱，執行 init 時將創建此目錄
# ksonnet應用程式將在 ${KFAPP}/ks_app目錄中創建
$ KFAPP=mykfapp
$ ${KUBEFLOW_REPO}/scripts/kfctl.sh init ${KFAPP} --platform none

# 安裝 Kubeflow 套件至應用程式目錄
$ cd ${KFAPP}
$ ${KUBEFLOW_REPO}/scripts/kfctl.sh generate k8s

# 定義 Namespace 為 kubeflow 來管理相關資源
$ NAMESPACE=kubeflow
$ kubectl create ns ${NAMESPACE}

# 部署 Kubeflow
$ ${KUBEFLOW_REPO}/scripts/kfctl.sh apply k8s
```

> #注意: 官方部署預設為 default，這邊部署有特別給 `kubeflow` 命名空間，如果要移除kubeflow，可直接 刪除kubeflow Namespace，所有物件也會被刪除
其中的KF_ENV為ksonnet定義的部署環境，這裡直接設為default

### 完成後檢查 Kubeflow 元件部署結果：
```bash=
$ kubectl -n kubeflow get pod -o wide
NAME                                                      READY     STATUS    RESTARTS   AGE       IP             NODE      NOMINATED NODE
ambassador-5cf8cd97d5-2tfqg                               1/1       Running   1          8m        10.244.2.167   k8s-g3    <none>
ambassador-5cf8cd97d5-4rbcx                               1/1       Running   1          8m        10.244.4.44    k8s-g1    <none>
ambassador-5cf8cd97d5-q6dnh                               1/1       Running   1          8m        10.244.3.104   k8s-g2    <none>
argo-ui-7c9c69d464-6hljq                                  1/1       Running   0          7m        10.244.2.171   k8s-g3    <none>
centraldashboard-6f47d694bd-bn55x                         1/1       Running   0          8m        10.244.3.106   k8s-g2    <none>
jupyter-0                                                 1/1       Running   0          8m        10.244.3.105   k8s-g2    <none>
katib-ui-6bdb7d76cc-bfd72                                 1/1       Running   0          6m        10.244.3.114   k8s-g2    <none>
metacontroller-0                                          1/1       Running   0          7m        10.244.3.109   k8s-g2    <none>
minio-7bfcc6c7b9-h9bw5                                    1/1       Running   0          7m        10.244.2.172   k8s-g3    <none>
ml-pipeline-6fdd759597-k5hds                              1/1       Running   1          7m        10.244.2.173   k8s-g3    <none>
ml-pipeline-persistenceagent-5669f69cdd-9tjdm             1/1       Running   0          6m        10.244.2.174   k8s-g3    <none>
ml-pipeline-scheduledworkflow-9f6d5d5b6-ddtbs             1/1       Running   0          6m        10.244.3.111   k8s-g2    <none>
ml-pipeline-ui-67f79b964d-brjlt                           1/1       Running   0          6m        10.244.3.112   k8s-g2    <none>
mysql-6f6b5f7b64-hbxpl                                    1/1       Running   0          7m        10.244.3.110   k8s-g2    <none>
pytorch-operator-6f87db67b7-d5hq4                         1/1       Running   0          7m        10.244.3.108   k8s-g2    <none>
studyjob-controller-774d45f695-wwznk                      1/1       Running   0          6m        10.244.3.115   k8s-g2    <none>
tf-job-dashboard-5f986cf99d-295kw                         1/1       Running   0          7m        10.244.2.168   k8s-g3    <none>
tf-job-operator-v1beta1-5876c48976-sk8gf                  1/1       Running   0          7m        10.244.3.107   k8s-g2    <none>
vizier-core-fc7969897-86ffs                               1/1       Running   0          6m        10.244.2.179   k8s-g3    <none>
vizier-core-rest-6fcd4665d9-rtpz6                         1/1       Running   0          6m        10.244.2.177   k8s-g3    <none>
vizier-db-777675b958-l966s                                1/1       Running   0          6m        10.244.2.178   k8s-g3    <none>
vizier-suggestion-bayesianoptimization-54db8d594f-lghgh   1/1       Running   0          6m        10.244.2.175   k8s-g3    <none>
vizier-suggestion-grid-6f5d9d647f-6pjt9                   1/1       Running   0          6m        10.244.2.176   k8s-g3    <none>
vizier-suggestion-hyperband-59dd9bb9bc-9knsd              1/1       Running   0          6m        10.244.3.113   k8s-g2    <none>
vizier-suggestion-random-6dd597c997-9xkrq                 1/1       Running   0          6m        10.244.2.180   k8s-g3    <none>
workflow-controller-5c95f95f58-ncv9d                      1/1       Running   0          7m        10.244.2.170   k8s-g3    <none>
```

### #補充 關閉 Kubeflow使用報告 (可選擇)
部署後會附加收集匿名用戶數據記錄，以協助改進 Kubeflow，這邊官方也提供[禁用說明](https://www.kubeflow.org/docs/other-guides/usage-reporting/)

```bash=
$ kubectl delete -n kubeflow deploy spartakus-volunteer
$ cd ../
$ cd ${KFAPP}/ks_app/
$ ks delete default  -c spartakus
$ ks component rm spartakus

...
INFO removing environment component                component-name=spartakus
INFO Removing component parameter references ...
INFO Deleting component "spartakus"
INFO Successfully deleted component 'spartakus'
```

---

### 訪問 Kubeflow Web UI

Kubeflow 提供了一系列 Web UI：

- Argo UI
- JupyterHub
- Tfjob Dashboard
- Katib Dashboard
- Pipeline Dashboard

首先，透過Service查看一下目前pod狀態

```bash=
$ kubectl -n kubeflow get svc -o wide -w
NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE       SELECTOR
ambassador                               ClusterIP   10.103.129.210   <none>        80/TCP              3m        service=ambassador
ambassador-admin                         ClusterIP   10.100.125.72    <none>        8877/TCP            3m        service=ambassador
argo-ui                                  NodePort    10.97.105.61     <none>        80:30965/TCP        2m        app=argo-ui
centraldashboard                         ClusterIP   10.98.13.76      <none>        80/TCP              3m        app=centraldashboard
jupyter-0                                ClusterIP   None             <none>        8000/TCP            3m        app=jupyter
jupyter-lb                               ClusterIP   10.108.219.1     <none>        80/TCP              3m        app=jupyter
katib-ui                                 ClusterIP   10.102.139.110   <none>        80/TCP              2m        app=vizier,component=ui
minio-service                            ClusterIP   10.105.24.96     <none>        9000/TCP            2m        app=minio
ml-pipeline                              ClusterIP   10.99.180.136    <none>        8888/TCP,8887/TCP   2m        app=ml-pipeline
ml-pipeline-tensorboard-ui               ClusterIP   10.103.12.204    <none>        80/TCP              2m        app=ml-pipeline-tensorboard-ui
ml-pipeline-ui                           ClusterIP   10.97.153.108    <none>        80/TCP              2m        app=ml-pipeline-ui
mysql                                    ClusterIP   10.108.186.1     <none>        3306/TCP            2m        app=mysql
tf-job-dashboard                         ClusterIP   10.104.211.110   <none>        80/TCP              3m        name=tf-job-dashboard
vizier-core                              NodePort    10.97.61.73      <none>        6789:30710/TCP      2m        app=vizier,component=core
vizier-core-rest                         ClusterIP   10.107.134.95    <none>        80/TCP              2m        app=vizier,component=core-rest
vizier-db                                ClusterIP   10.98.2.75       <none>        3306/TCP            2m        app=vizier,component=db
vizier-suggestion-bayesianoptimization   ClusterIP   10.104.57.93     <none>        6789/TCP            2m        app=vizier,component=suggestion-bayesianoptimization
vizier-suggestion-grid                   ClusterIP   10.111.0.101     <none>        6789/TCP            2m        app=vizier,component=suggestion-grid
vizier-suggestion-hyperband              ClusterIP   10.100.181.37    <none>        6789/TCP            2m        app=vizier,component=suggestion-hyperband
vizier-suggestion-random                 ClusterIP   10.101.159.47    <none>        6789/TCP            2m        app=vizier,component=suggestion-random
```

這時候就可以透過 `Central UI` 引導訪問不同的 Kubeflow 服務，由於暴露安全性問題，官方教學是透過本機使用 [port forward](https://www.kubeflow.org/docs/other-guides/accessing-uis/) 訪問，但這邊為實驗測試是直接修改 Kubernetes Service，透過`NodePort`來直接連線：


```bash=
$ kubectl -n kubeflow edit svc ambassador

sessionAffinity: None
  type: NodePort
status:
```
再次查看Service確認NodePort 對應 Port
```bash=
$ kubectl -n kubeflow get svc -o wide
NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)             AGE       SELECTOR
ambassador                               NodePort    10.103.129.210   <none>        80:32078/TCP        14m       service=ambassador
```
完成後連接 `http://Master_IP:Port` 即可看到 kubeflow Central UI

![](https://i.imgur.com/PEOizuf.png)

--- 

## 測試 Jupyter Notebook

Kubeflow v0.4 更新 JupyterHub UI介面，創建Jupyter Notebook 比之前配置更容易。可以選擇已安裝最新版本的 ML Images，或附上自己自定義Image，即可配置`Spawn`生成。

這邊需要再次先建立一個 `NFS PV `來提供給 Kubeflow Jupyter使用：

```bash=
$ cat <<EOF | kubectl create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: jupyter-notebook-pv
spec:
  capacity:
    storage: 20Gi
  accessModes:
    - ReadWriteOnce
  nfs:
    server: {NFS-server IP}
    path: /nfs-data/jupytertest
EOF
```
`NFS PV` 建立好後這邊 UI 點選Jupyter Notebook登入測試，並輸入任意`帳號密碼`進行登入：

![](https://i.imgur.com/zkIWIkP.png)

**Spawner Optinos** 這邊可以輸入Notebook 資源，包含 Image、CPU、Memory、GPU數量、Workspace Volume、Data Volumes，最後點選Spawn來完成建立 Server，如下圖所示：

![](https://i.imgur.com/4KRHrPA.png)


![](https://i.imgur.com/NywKU10.png)

配置叢集節點GPU分配需求
![](https://i.imgur.com/PrKHHr8.png)

配置好後及快速生成一個乾淨並帶有GPU使用的 **Jupyter Notebook**
![](https://i.imgur.com/7iiyIys.png)


## Kubeflow Pipelines UI (待補)

Kubeflow主要是為了簡化在Kubernetes上面運行機器學習任務的流程，最終希望能夠實現一套完整工作流(workflow)模型。所謂工作流，或者稱之為流水線(pipelines)，來實現 ML 從數據到模型的一整套 End-to-End 的過程 : 


![](https://i.imgur.com/shEDXxm.png)
![](https://i.imgur.com/tEocDzK.png)

### # 相關文章（補充)
-  [Kubeflow Pipelines: 面向机器学习场景的流水线系统的使用与实现](http://gaocegege.com/Blog/kfp)
-  [再談Kubeflow - pipline](https://soolaugust.github.io/learn-k8s-from-scratch/kubeflow-intro.html)

---

## Summary

此篇文章介紹 kubeflow v0.4 發佈資訊更新資訊與部署流程，但由於 kubeflow 開源專案更新速度太快，要對於每項服務都測試需要花時間慢慢整理，至於 v0.4 針對 pipline 核心組件值得花時間研究，後續也會介紹更多[Components](https://www.kubeflow.org/docs/components/)應用操作。

> 近日開始協助貢獻更新 `kubeflow/website` 範例與錯誤資訊，希望能為 kubeflow 做更多的推廣

## Reference

-  [Kubeflow 0.4 Release: Enhancements for Machine Learning productivity
Go to the profile of Thea Lamkin](https://medium.com/kubeflow/kubeflow-0-4-release-enhancements-for-machine-learning-productivity-d77c54df07a9)
- [新Kubeflow，新征程（一）：簡化部署體驗](https://yq.aliyun.com/articles/686672)
