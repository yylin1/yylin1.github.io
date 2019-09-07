---
title: 如何更換 kubeflow 預設 Scheduling
toc: true 
date: 2019-01-22 23:34:29
categories: 
- Kubeflow
tags: 
- kubernetes
- kubeflow
- scheduler
thumbnail: /images/kubeflow-job-scheduling.png
---

本篇文章將說明實作官網 [[kubeflow/Job Scheduling]](https://www.kubeflow.org/docs/other-guides/job-scheduling/)，所介紹的如何使用 `gang-scheduling` 更換原 kubeflow 預設 scheduler 機制。文章教學介紹透過使用[kube-batch](https://github.com/kubernetes-sigs/kube-batch)以支持 kubeflow 透過`gang-scheduling`排程方式，來同時允許多個Pod執行解決`deadlock`問題。

> **tf-operator 實現 gang-scheduler 已經從 `PDB` 轉換為 `PodGroups`已更新於文章** 
> 請參考 [Kube-batch-tutorial](https://github.com/kubernetes-sigs/kube-batch/blob/master/doc/usage/tutorial.md) 介紹 or [volcano](https://github.com/volcano-sh/volcano)

<!-- more -->

**本篇實作主要感謝 @kubeflow member and Kubeflow/tf-operator reviewer [@Jack Lin](https://github.com/ChanYiLin) 撰寫的教學**

## 實驗環境

本實驗環境為 kubernetes HA Cluster + 4*GPU Node :

- Kubernetes v1.11.2
- Etcd v3.2.9
- containerd v1.1.2


| IP Address| Hostname | CPU | Memory | Extra Device|
| --------  | -------- | -------- |-------- |-------- |
| 192.168.0.98  |VIP	|	|   |     | 
| 192.168.0.81 | k8s-m1 | 4     | 16G |  無
| 192.168.0.82 | k8s-m2 | 4     | 16G |  無
| 192.168.0.83 | k8s-m3 | 4     | 16G |  無
| 192.168.0.84 | k8s-g1 | 4     | 16G | GTX 1060 6G
| 192.168.0.85 | k8s-g2 | 4     | 16G | GTX 1060 6G
| 192.168.0.86 | k8s-g3 | 4     | 16G | GTX 1060 6G
| 192.168.0.87 | k8s-g4 | 4     | 16G | GTX 1060 6G

--- 

## 配置更換預設 scheduling

要讓環境使用`gang-scheduling`排程，必須先在叢集中安裝 [kube-batch](https://github.com/kubernetes-sigs/kube-batch) 作為 Kubernetes 額外可選擇的排程機制，並配置 operator 以啟用`gang-scheduling`。

#### Note by job scheduling :

> - Kube-batch’s introduction is here, and also check how to install it here
> - Take tf-operator for example, enable gang-scheduling in tf-operator by setting true to `--enable-gang-scheduling` flag.


### 1. kube-batch install 

參考`kube-batch` 詳細介紹與安裝方式[[tutorial]](https://github.com/kubernetes-sigs/kube-batch/blob/master/doc/usage/tutorial.md)，以下為實驗記錄。


在`叢集中` 安裝 Helm tool：

```bash=
$ wget -qO- https://storage.googleapis.com/kubernetes-helm/helm-v2.12.2-linux-amd64.tar.gz | tar -zx
$ sudo mv linux-amd64/helm /usr/local/bin/
```

初始化 Helm (並安裝 Tiller Server):

```bash=
$ kubectl -n kube-system create sa tiller
$ kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
$ helm init --service-account tiller

Tiller (the Helm server-side component) has been installed into your Kubernetes Cluster.
Happy Helming!
```

###  # Config kube-batch for Kubernetes

在`k8s-m1`節點上執行：

- Download kube-batch

```bash=
$ mkdir -p $GOPATH/src/github.com/kubernetes-sigs/
$ cd $GOPATH/src/github.com/kubernetes-sigs/
$ git clone http://github.com/kubernetes-sigs/kube-batch
```

- Deploy kube-batch by Helm

```bash=
$ helm install $GOPATH/src/github.com/kubernetes-sigs/kube-batch/deployment/kube-batch --namespace kube-system
NAME:   mortal-buffalo
LAST DEPLOYED: Sun Jan 20 19:12:50 2019
NAMESPACE: kube-system
STATUS: DEPLOYED

RESOURCES:
==> v1beta1/CustomResourceDefinition
NAME                                   AGE
podgroups.scheduling.incubator.k8s.io  0s
queues.scheduling.incubator.k8s.io     0s

==> v1/Deployment
kube-batch  0s

==> v1/Pod(related)

NAME                         READY  STATUS   RESTARTS  AGE
kube-batch-5d7d955f84-jw8gx  0/1    Pending  0         0s


NOTES:
The batch scheduler of Kubernetes.
```

驗證 kube-batch 已部署於kubernetes：

```bash=
$ helm list
NAME           	REVISION	UPDATED                 	STATUS  	CHART           	APP VERSION	NAMESPACE
zooming-cricket	1       	Wed May 15 15:07:04 2019	DEPLOYED	kube-batch-0.4.2	           	kube-system
```

---

### 2. 新增 RBAC 給 kube-batch 

GitHub 安裝 `kube-batch` 補充說明提到，需要自己額配置RBAC給`kube-bath`

#### Note by kube-batch : 

> - kube-batch need to collect cluster information(such as Pod, Node, CRD, etc) for scheduling, so the service account used by the deployment must have permission to access those cluster resources, otherwise, kube-batch will fail to startup. 
> - For users who are not familiar with Kubernetes RBAC, please copy the `example/role.yaml` into `$GOPATH/src/github.com/kubernetes-sigs/kube-batch/deployment/kube-batch/templates/` and reinstall batch.

創建 RBAC 「ServiceAccount」、「ClusterRoleBinding」，名稱並為`name: kube-batch`: 

```bash=
$ cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: default-sa-admin
subjects:
  - kind: ServiceAccount
    name: default
    namespace: kube-system
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
EOF
```
---
### 3. kube-batch 啟用 gang-scheduling 配置

確認`叢集中` 所配置的 kubeflow Namespace 當前運行的 Deploment，並找到`tf-job-operator`: 

- 將以下規則添加到 tf-job-operator clusterrole 以授予 tf-job-operator 創建權限 `PodGroup`

```bash=
$ kubectl -n kubeflow edit clusterrole  tf-job-operator
...
...
- apiGroups:
  - scheduling.incubator.k8s.io
  resources:
  - podgroups
  verbs:
  - '*'
```


- tf-operator 通過添加標記啟用 gang-scheduling `--enable-gang-scheduling=true`

```bash=
$ kubectl -n kubeflow edit deployment tf-job-operator
...
spec:
      containers:
      - command:
        - /opt/kubeflow/tf-operator.v1beta2
        - --alsologtostderr
        - -v=1
        - --enable-gang-scheduling=true
...
```

### 4. 測試範例 - 配置 YAML 增加 PodGroups 

這邊要注意運行的範例 TFJob，需要配置對應 `scheduling.k8s.io/group-name: ${PodGroupName}` 已啟用 PodGroups，而名稱要與當前 Job `${name}`設定相同。

```bash=
apiVersion: "kubeflow.org/v1beta1"
kind: "TFJob"
metadata:
  name: "gang-scheduling-podgroup"
spec:
  cleanPodPolicy: None
  tfReplicaSpecs:
    Worker:
      replicas: 2
      restartPolicy: Never
      template:
        metadata:
          annotations:
            scheduling.k8s.io/group-name: "gang-scheduling-podgroup"
        spec:
          schedulerName: kube-batch
          containers:
            - name: tensorflow
              image: gcr.io/kubeflow-examples/distributed_worker:v20181031-513e107c
              resources:
                limits:
                  nvidia.com/gpu: 1
```



### [補充說明] TFJob 配置 YAML 實現 gang-scheduling

執行 [job-scheduling](https://www.kubeflow.org/docs/other-guides/job-scheduling/) 介紹的範例：

```bash=
apiVersion: "kubeflow.org/v1beta1"
kind: "TFJob"
metadata:
  name: "tfjob-gang-scheduling"
  namespace: "kubeflow"
spec:
  tfReplicaSpecs:
    Worker:
      replicas: 1
      template:
        metadata:
          annotations:
            scheduling.k8s.io/group-name: "tfjob-gang-scheduling"
        spec:
          schedulerName: kube-batch
          nodeSelector:
             gpu: "1060g1"
          containers:
          - args:
            - python
            - tf_cnn_benchmarks.py
            - --batch_size=4
            - --model=inception3
            - --variable_update=parameter_server
            - --num_gpus=1
            - --local_parameter_device=cpu
            - --device=gpu
            - --data_format=NHWC
            image: yylin1/tf-benchmarks-gpu:gpu-v1
            name: tensorflow
            resources:
              limits:
                nvidia.com/gpu: 1
            workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
    PS:
      replicas: 1
      template:
        metadata:
          annotations:
            scheduling.k8s.io/group-name: "tfjob-gang-scheduling"
        spec:
          schedulerName: kube-batch
          nodeSelector:
             gpu: "1060g1"
          containers:
          - args:
            - python
            - tf_cnn_benchmarks.py
            - --batch_size=4
            - --model=inception3
            - --variable_update=parameter_server
            - --num_gpus=1
            - --local_parameter_device=cpu
            - --device=cpu
            - --data_format=NHWC
            image: yylin1/tf-benchmarks-gpu:gpu-v1
            name: tensorflow
            resources:
              limits:
                cpu: '1'
            workingDir: /opt/tf-benchmarks/scripts/tf_cnn_benchmarks
          restartPolicy: OnFailure
```

- `tfjob-gang-scheduling` yaml範例中，要使用kube-batch為選用的排程機制，這邊要在Spec指定 `schedulerName:kube-batch`。


```bash=
 spec:
          schedulerName: kube-batch
```

---

## [補充說明] gang-scheduling 實驗測試

環境目前有`4個GPU Node節點`，運行流程先執行`yylin-job1`之後再執行`yylin-job2` 測試環境，以觀察:

- 通過使用`kube-batch`來啟用`gang-scheduling`，而只有當有叢集環境中有足夠的資源執行給所有pod時，才能運行任務。否則所有pod將處於`Pending`狀態，等待足夠的資源。
- 若使用預設`kubeflow scheuler`會導致同一個Job資源不夠，但Worker照常運行，導致執行Job任務是無意義的。

Job 配置：
- yylin-job1 
    - Worker:3、PS:1
- yylin-job2
    - Worker:2、PS:1


啟用`schedulerName: kube-batch`:

```bash=
$ kubectl get pod -n kubeflow
tfjob-gang-scheduling-ps-0                                1/1       Running             0          43s
tfjob-gang-scheduling-worker-0                            1/1       Running             0          43s
tfjob-gang-scheduling-worker-1                            1/1       Running             0          43s
tfjob-gang-scheduling-worker-2                            1/1       Running             0          43s
tfjob-gang-scheduling2-ps-0                               1/1       Running             0          38s
tfjob-gang-scheduling2-worker-0                           0/1       Pending             0          38s
tfjob-gang-scheduling2-worker-1                           0/1       Pending             0          38s
```

預設`kubeflow scheuler`:

```bash=
$ kubectl get pod -n kubeflow
tfjob-gang-scheduling-ps-0                                1/1       Running             0          1m
tfjob-gang-scheduling-worker-0                            1/1       Running             0          1m
tfjob-gang-scheduling-worker-1                            1/1       Running             0          1m
tfjob-gang-scheduling-worker-2                            1/1       Running             0          1m
tfjob-gang-scheduling2-ps-0                               1/1       Running             0          34s
tfjob-gang-scheduling2-worker-0                           1/1       Running             0          34s
tfjob-gang-scheduling2-worker-1                           0/1       Pending             0          34s
```

Reference: 
- [[kubeflow] [Job Scheduling- schedule a job with gang-scheduling]](https://www.kubeflow.org/docs/other-guides/job-scheduling//)
