---
title: 認識 Bastion Host 部署管理機
toc: true
date: 2019-01-19 23:51:36
categories: 
- Deploy
tags:
- bastion
thumbnail: /images/bastion-host.png
---

由於最近接觸 `Openshift` 部署實體機，剛好部署過程學習到透過 `Bastion 管理機`，來統一管理分配對應依賴套件安裝給對應節點，並用於運行 Ansible 腳本及配置叢集使用，但後來思考到整體部署過程中`安全性`問題，所以爬文搜尋一下大部分進行管理機部署情境與針對安全原則筆記。

<!-- more -->

## Bastion Host 中繼管理機

> 簡單來說就是為連線伺服器叢集限定一個入口，方便管理部署維運並權限控制以及監控

想像一下，如果你要維護一百多台伺服器，要記住每台伺服器IP/帳號/密碼，肯定是不可能的吧？這種情況下，可以在跳板機上配置通常做法將`SSH Key`複製到遠端 Server 再進行跳板動作，配置完畢後，通過跳板機登錄並操作維護多台節點主機。

但因為目前我的環境都為測試實驗環境，這做法其實沒有很安全，又需要多下一個指令進透過中繼管理機操作，而先前更何況沒有`Bastion Host`概念通常都直接拿`Master Node`當跳板機部署。而更重要的是當`Bastion Host`跳板機被 Hack 都可以隨時入侵其他主機操作，而找如何針對增加安全性的問題下手。


## 解決安全性辦法

> 這邊參考安全性做法 
> - [[Ref:安全地利用 Bastion Host 跳轉]](https://medium.com/@puresmash/%E5%AE%89%E5%85%A8%E5%9C%B0%E5%88%A9%E7%94%A8-bastion-host-%E8%B7%B3%E8%BD%89-5f0529301362)
> - [[Ref:SSH Agent Forwarding 教學]](https://blog.wu-boy.com/2016/10/ssh-agent-forwarding-proxycommand-tutorial/)

- 針對「 Bastion Host 中繼主機當跳板」連線白名單限制，或是直接隔絕外網連線操作，只在限定實驗環境內網進行中繼主機操作與部署（離線部署叢集）

轉交密鑰方式：
- [SSH agent forwarding](https://developer.github.com/guides/using-ssh-agent-forwarding/) 讓開發者將 Local 端的 SSH Key Pair 帶到另外一台機器進行傳送，也就是說你不用將 SSH Key 複製到遠端 Server 再進行跳板動作

舉例來說，首先 SSH 登入進 Server1，接著在 Server1 上登入 Server2 時，就會自動使用你本地的 SSH Key：

```
Local ---(SSH)---> Server1 ---(SSH)---> Server2
```

首先需要將要 `forwarding` 的 `private key` 加到清單裡面:

```bash=
$ ssh-add
```
加上 `-L` 可以檢查這個清單

```bash=
$ ssh-add -L
```

這邊注意可以將所有機器的 key pair 都放到 ~/.ssh/keys 目錄，並且設定 400 權限

方法一：每次要 Local 連上 Server 時，加上 -A 參數：

```bash=
$ ssh -A user@{you server1 ip}
```

方法二：修改 `~/.ssh/config` 設定，加上 `ForwardAgent yes`，這樣就不需要每次連都加上 -A 參數。例如

```bash=
Host server1
  HostName 106.187.36.122
  ForwardAgent yes
```

- 這樣中繼主機，不需要存放任何憑證資料

> 安全隱憂的，起因於 agent service 於連線時會在 /tmp/ 目錄建立一個檔案 /tmp/ssh-xxxxxx/agent.xxxx，在正常的情況下，它會用原使用者透過 ssh-add 所設定的密鑰對，替我們回答目的伺服器所發起的密鑰驗證要求。

![](https://cdn-images-1.medium.com/max/1600/1*HLs0qwn9LdoedmIlQ8VlVw.png)


```bash=
Host hosta
  User userfoo
  Hostname 123.123.123.123
 
Host hostb
  User userbar
  Hostname 192.168.1.1
  Port 22
  ProxyCommand ssh -q -W %h:%p hosta
  IdentityFile ~/.ssh/keys/xxxx.pem
``` 
此配置告訴本地SSH客戶端通過直接連接連接到主機A. 現在，如果你鍵入 ssh hostB 它將要做的是首先連接到 主機A 然後通過參數 ProxyCommand 指定的主機和 Port 轉發你的SSH客戶端-W，在這種情況下 `192.168.1.1` 強調這是一個內部主機。然後，您的本地 SSH 客戶端將對 主機B 進行身份驗證，就像您直接連接到它一樣。

這裡的另一個觀察是你不需要記住關於主機的任何信息，當你可以簡單地用Host線對它進行別名時，它不需要IP或模糊的主機名，端口或用戶名。

ProxyCommand 不用將 pem 透過 ssh-add 匯入，省下一步指令，接著執行底下指令就可以 ssh 到指定機器，跟方法一比起來更安全又快速

```bash=
$ ssh hostb
```

這個方法可以讓我們直接登入遠端主機，省略第二次的 ssh 指令。
