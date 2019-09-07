---
title: Portainer 透過 TLS 認證
toc: true
date: 2019-01-28 22:46:42
categories: 
- Docker
tags: 
- docker
- UI
thumbnail: /images/portainer-tls.png
---

Portainer 是一個輕量級的 UI 管理界面，可輕鬆管理 Docker 主機或 Swarm 叢集，配置遠端管理會使用到Docker API，而預設都會開啟沒有安全保護的 `port 2375` 產生連線漏洞，而我們就會需要配置TLS認證憑證，以確保連線安全，此篇會說明如何針對`新增Endpoints主機`配置加入TLS加密。

<!-- more -->

## Portainer概述 

Simple management UI for Docker. [Deploy Portainer](https://portainer.readthedocs.io/en/latest/deployment.html) & [Documentation](https://portainer.readthedocs.io/)

- Portainer 使用非常簡單。它可以在任何Docker引擎上運行的單個Container（Docker for Linux和Docker for Windows）

- Portainer允許管理Docker containers, images, volumes, networks等。它與獨立的Docker引擎和Docker Swarm兼容。

![](https://i.imgur.com/v1Fx5WR.png)

![](https://d1jiktx90t87hr.cloudfront.net/354/wp-content/uploads/sites/2/2018/12/endpoints1.png)
---

### 版本限制

Portainer 支持以下Docker版本：

- Docker 1.10 to the latest version
- Standalone Docker Swarm >= 1.2.3

---

## 部署 Portainer

> 官網安裝方法可[參考](https://portainer.io/install.html) 

```bash=
$ docker image pull portainer/portainer
$ docker run -d -p 9000:9000 --name portainer --restart always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer
```
創建後即可查看container狀態
```bash=
$ docker ps
CONTAINER ID        IMAGE                 COMMAND             CREATED             STATUS              PORTS                    NAMES
04811d58f2bc        portainer/portainer   "/portainer"        26 hours ago        Up 26 hours         0.0.0.0:9000->9000/tcp   portainer
```

## 快速生成 TLS 憑證 

- 官網配置教學參考: [Protect the Docker daemon socket](https://docs.docker.com/engine/security/https/)


由於Docker官網配置細節複雜，找很久相關生成方法，最後找到別人撰寫好的TLS生成`shell執行檔`，能快速利用 `openssl` 生成相應的`CA key`憑證。

> [Ref: [CoreOS配置Docker API TLS認證]](https://my.oschina.net/ykbj/blog/1920021)

這邊直接創建以下`shell`執行檔生成對應 TLS 憑證`CA key`，執行配置請依照`當前環境`修改配置後進行執行`shell`：

```bash=
$ vim auto-tls-certs.sh

#!/bin/bash
# 
# -------------------------------------------------------------
# 自動創建 Docker TLS 憑證
# -------------------------------------------------------------

# 以下是配置信息
# --[BEGIN]------------------------------

CODE="辨識生成key名稱"  
IP="節點IP"
PASSWORD="憑證密碼"
COUNTRY="TW"
STATE="Taipei"
CITY="Taipei"
ORGANIZATION="公司"
ORGANIZATIONAL_UNIT="Dev"
COMMON_NAME="$IP"
EMAIL="service@input.email"

# --[END]--

# Generate CA key
openssl genrsa -aes256 -passout "pass:$PASSWORD" -out "ca-key-$CODE.pem" 4096
# Generate CA
openssl req -new -x509 -days 365 -key "ca-key-$CODE.pem" -sha256 -out "ca-$CODE.pem" -passin "pass:$PASSWORD" -subj "/C=$COUNTRY/ST=$STATE/L=$CITY/O=$ORGANIZATION/OU=$ORGANIZATIONAL_UNIT/CN=$COMMON_NAME/emailAddress=$EMAIL"
# Generate Server key
openssl genrsa -out "server-key-$CODE.pem" 4096

# Generate Server Certs.
openssl req -subj "/CN=$COMMON_NAME" -sha256 -new -key "server-key-$CODE.pem" -out server.csr

echo "subjectAltName = IP:$IP,IP:127.0.0.1" >> extfile.cnf
echo "extendedKeyUsage = serverAuth" >> extfile.cnf

openssl x509 -req -days 365 -sha256 -in server.csr -passin "pass:$PASSWORD" -CA "ca-$CODE.pem" -CAkey "ca-key-$CODE.pem" -CAcreateserial -out "server-cert-$CODE.pem" -extfile extfile.cnf


# Generate Client Certs.
rm -f extfile.cnf

openssl genrsa -out "key-$CODE.pem" 4096
openssl req -subj '/CN=client' -new -key "key-$CODE.pem" -out client.csr
echo extendedKeyUsage = clientAuth >> extfile.cnf
openssl x509 -req -days 365 -sha256 -in client.csr -passin "pass:$PASSWORD" -CA "ca-$CODE.pem" -CAkey "ca-key-$CODE.pem" -CAcreateserial -out "cert-$CODE.pem" -extfile extfile.cnf

rm -vf client.csr server.csr

chmod -v 0400 "ca-key-$CODE.pem" "key-$CODE.pem" "server-key-$CODE.pem"
chmod -v 0444 "ca-$CODE.pem" "server-cert-$CODE.pem" "cert-$CODE.pem"

# 打包 client 憑證
mkdir -p "tls-client-certs-$CODE"
cp -f "ca-$CODE.pem" "cert-$CODE.pem" "key-$CODE.pem" "tls-client-certs-$CODE/"
cd "tls-client-certs-$CODE"
tar zcf "tls-client-certs-$CODE.tar.gz" *
mv "tls-client-certs-$CODE.tar.gz" ../
cd ..
rm -rf "tls-client-certs-$CODE"

# 拷貝服務憑證
mkdir -p /etc/docker/certs.d
cp "ca-$CODE.pem" "server-cert-$CODE.pem" "server-key-$CODE.pem" /etc/docker/certs.d/
```

腳本執行完成後顯示訊息：
```bash=
$ ./create_ca_key.sh 

Generating RSA private key, 4096 bit long modulus
......................................................++
.........................................................................................................................++
e is 65537 (0x10001)
Generating RSA private key, 4096 bit long modulus
......................................................++
..............++
e is 65537 (0x10001)
Signature ok
subject=/CN=140.128.18.88
Getting CA Private Key
Generating RSA private key, 4096 bit long modulus
....................................................................................................................++
..............................................++
e is 65537 (0x10001)
Signature ok
subject=/CN=client
Getting CA Private Key
removed 'client.csr'
removed 'server.csr'
mode of 'ca-key-s1.pem' changed from 0644 (rw-r--r--) to 0400 (r--------)
mode of 'key-s1.pem' changed from 0644 (rw-r--r--) to 0400 (r--------)
mode of 'server-key-s1.pem' changed from 0644 (rw-r--r--) to 0400 (r--------)
mode of 'ca-s1.pem' changed from 0644 (rw-r--r--) to 0444 (r--r--r--)
mode of 'server-cert-s1.pem' changed from 0644 (rw-r--r--) to 0444 (r--r--r--)
mode of 'cert-s1.pem' changed from 0644 (rw-r--r--) to 0444 (r--r--r--)
```
對腳本中配置參數進行修改後運行，自動會生成TLS憑證，在/etc/docker/certs.d/目錄下:

```bash=
$ ls /etc/docker/certs.d/
ca-dp.pem  ca-s1.pem  server-cert-dp.pem  server-cert-s1.pem  server-key-dp.pem  server-key-s1.pem
```

其中運行`shell腳本`的路徑下，同時還會自動壓縮好了一個`.tar.gz`檔，方便我們認證使用。

```bash=
$ ls
tls-client-certs-s1.tar.gz ca-key-s1.pem  ca-s1.pem  ca-s1.srl  cert-s1.pem  create_ca_key.sh  extfile.cnf  key-s1.pem  server-cert-s1.pem  server-key-s1.pem  
```

這邊`Portainer UI` 直接透過`tls-client-certs-{CODE}.tar.gz`這個壓縮檔裡面`CA Key`進行 `Portainer TLS `認證。

---

## 新增 Endpoints 主機並配置TLS

在使用 `portainer` 管理Docker主機，為了安全性關係新增 Endpoints 時，需要開啟 TLS 認證，上傳 docker-mahcine 部署主機的憑證和key：

- `TLS CA certificate`：ca.pem
- `TLS certificate`：cert.pem
- `TLS key`：key.pem


### 開啟Remote API訪問

首先確認，TLS 生成出來憑證 `ca.pem`,`cert.pem`,`key.pem` 路徑，環境實驗路徑`/root/private/`。


#### 修改docker啟動文件
```bash=
$ vim  /lib/systemd/system/docker.service
```
#### 修改ExecStart配置參數

- 新增`-H tcp://0.0.0.0:2375 -H unix://var/run/docker.sock`
- 依照路徑對應 `ca.pem`, `cert.pem`, `key.pem` 配置

```bash=
ExecStart=/usr/bin/dockerd  -H unix://var/run/docker.sock -D -H tcp://0.0.0.0:2375 --tlsverify --tlscacert=/root/ca-dp.pem --tlscert=/root/server-cert-dp.pem --tlskey=/root/server-key-dp.pem
```
#### portainer 指令說明

> - --tlsverify: TLS support (default: false)
> --tlscacert: Path to the CA (default: /certs/ca.pem on Linux）
> --tlscert: Path to the TLS certificate file (default: /certs/cert.pem）
> --tlskey: Path to the TLS key (default: /certs/key.pem, 

配置好後重啟 Docker
```bash=
$ systemctl daemon-reload

重新啟動 docker
$ systemctl restart  docker

檢查 參數
$ ps -ef  | grep docker

$ ps -ef  | grep docker
root     18434     1  2 16:54 ?        00:00:00 /usr/bin/dockerd -H unix://var/run/docker.sock -D -H tcp://0.0.0.0:2375 --tlsverify --tlscacert=/root/private/ca-s1.pem --tlscert=/root/private/server-cert-s1.pem --tlskey=/root/private/server-key-s1.pem
root     18558 15435  0 16:54 pts/0    00:00:00 grep --color=auto docker

```

#### #補充: 測試TLS認證

將`cert.pem`,`key.pem`這兩個檔案複製到測試機上,透過 curl中`-k`意思是(Allow connections to SSL sites without certs)驗證憑證
  
```bash=
$ curl -k https://{docker節點IP}:2375/info --cert ./cert.pem --key ./key.pem
```

返回docker信息即為完成憑證認證

```bash=
$ curl -k https://140.128.18.88:2375/info --cert cert-dp.pem --key key-dp.pem 
{"ID":"F2HY:7GDT:YA3J:74UZ:2AKP:GW2Z:MIIT:GPG3:NOHZ:EKCW:ZFCA:M575","Containers":2,"ContainersRunning":1,"ContainersPaused":0,"ContainersStopped":1,"Images":22,"Driver":"overlay2","DriverStatus":[["Backing Filesystem","extfs"],["Supports d_type","true"],["Native Overlay Diff","true"]],"SystemStatus":null,"Plugins":{"Volume":["local"],"Network":["bridge","host","macvlan","null","overlay"],"Authorization":null,"Log":["awslogs","fluentd","gcplogs","gelf","journald","json-file","local","logentries","splunk","syslog"]},"MemoryLimit":true,"SwapLimit":false,"KernelMemory":true,"CpuCfsPeriod":true,"CpuCfsQuota":true,"CPUShares":true,"CPUSet":true,"IPv4Forwarding":true,"BridgeNfIptables":true,"BridgeNfIp6tables":true,"Debug":true,"NFd":31,"OomKillDisable":true,"NGoroutines":44,"SystemTime":"2019-01-28T15:26:15.584473249+08:00","LoggingDriver":"json-file","CgroupDriver":"cgroupfs","NEventsListener":0,"KernelVersion":"4.13.0-36-generic","OperatingSystem":"Ubuntu 16.04.4 LTS","OSType":"linux","Architecture":"x86_64","IndexServerAddress":"https://index.docker.io/v1/","RegistryConfig":{"AllowNondistributableArtifactsCIDRs":[],"AllowNondistributableArtifactsHostnames":[],"InsecureRegistryCIDRs":["127.0.0.0/8"],"IndexConfigs":{"docker.io":{"Name":"docker.io","Mirrors":[],"Secure":true,"Official":true}},"Mirrors":[]},"NCPU":4,"MemTotal":16739332096,"GenericResources":null,"DockerRootDir":"/var/lib/docker","HttpProxy":"","HttpsProxy":"","NoProxy":"","Name":"lab506-s1","Labels":[],"ExperimentalBuild":false,"ServerVersion":"18.09.1","ClusterStore":"","ClusterAdvertise":"","Runtimes":{"nvidia":{"path":"nvidia-container-runtime"},"runc":{"path":"runc"}},"DefaultRuntime":"runc","Swarm":{"NodeID":"","NodeAddr":"","LocalNodeState":"inactive","ControlAvailable":false,"Error":"","RemoteManagers":null},"LiveRestoreEnabled":false,"Isolation":"","InitBinary":"docker-init","ContainerdCommit":{"ID":"9754871865f7fe2f4e74d43e2fc7ccd237edcbce","Expected":"9754871865f7fe2f4e74d43e2fc7ccd237edcbce"},"RuncCommit":{"ID":"96ec2177ae841256168fcf76954f7177af9446eb","Expected":"96ec2177ae841256168fcf76954f7177af9446eb"},"InitCommit":{"ID":"fec3683","Expected":"fec3683"},"SecurityOptions":["name=apparmor","name=seccomp,profile=default"],"ProductLicense":"Community Engine","Warnings":["WARNING: No swap limit support"]}
```

### 完成後配置遠端節點 Endpoints 

Portainer透過上傳CA key，這邊直接解壓縮`shell`生成`tls-client-certs-{CODE}.tar.gz`的壓縮檔直接加入對應的TLS。


![](https://i.imgur.com/Ip80daq.png)

---

## 設置 docker默認 runtime 為 nvidia

由於環境Portainer預設執行為`docker runtime`，導致如果啟動Nvidia-Docker所執行的GPU Container會無法使用，這邊 Docker default runtime 改成 nvidia。

新增`"default-runtime": "nvidia"`指派配置，直接用docker取代nvidia-docker。
```bash=
$ vim /etc/docker/daemon.json

{
    "default-runtime": "nvidia",
    "runtimes": {
        "nvidia": {
            "path": "/usr/bin/nvidia-container-runtime",
            "runtimeArgs": []
        }
    }
}
```

完成後儲存，並重新啟動 Docker：
```bash=
$ sudo systemctl daemon-reload && sudo systemctl restart docker
```

Portainer 即可正常創建GPU Container: 

![](https://i.imgur.com/4bddhEY.png)

![](https://i.imgur.com/KGj02Bu.png)


---
## Related Posts

- [[Portainer with openSUSE Leap 15 小記]](http://sakananote2.blogspot.com/2018/06/portainer-with-opensuse-leap-15.html)
- [[CoreOS配置Docker API TLS认证]]](https://my.oschina.net/ykbj/blog/1920021)
- [[ docker 開啟remote api訪問，並使用TLS加密]](https://hk.saowen.com/a/5e0307008b31189c6923c4ede3a8fca83742cb1dd63a57c0fd4cf72233c791b2)


