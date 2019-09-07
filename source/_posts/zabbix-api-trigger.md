---
title: 透過 Zabbix API 監測 Trigger 狀態
toc: true
date: 2018-11-29 23:55:00
categories: 
- Zabbix
tags:
- zabbix
- API
thumbnail: /images/zabbix_trigger.png
---

此篇主要說明如何透過，Zabbix 提供的觸發器(Trigger)來監測 CPU 超標狀態，並透過 `python` 直接呼叫Zabbix API獲取所需要的狀態資訊，並協助監測遷移。

<!-- more -->
## 1. 新建Zabbix Trigger觸發器

本節你會學習如何配置一個觸發器（trigger）

首先登入你Zabbix網頁並輸入帳號密碼

範例：
```bash=
http://localhost/zabbix/
```

進入Zabbix網站後，開始配置觸發器（trigger）

* 前往配置（Configuration）→ 主機（Hosts）→ 選擇你要進行配置觸發器（trigger）的節點
> 右側可以選擇要設置的環境，這邊`Group`我們選擇`Compute`節點群

![](https://i.imgur.com/xX8gHUR.png)

* 接續會顯示`Compute`節點中可以觀察的三台Node主機 → 這邊我們直接點選`node-1.domain.tld`預備測試的節點進入設置

* 進入節點點擊上方的觸發器（Triggers），然後即可點擊創建觸發器（Create trigger），這將會向我們顯示一個觸發器定義表單

![](https://i.imgur.com/v9L91MO.jpg)

* 找到'新增主機（New host）'，點擊旁邊的觸發器（Triggers），然後點擊創建觸發器（Create trigger）。這將會向我們展現一個觸發器定義表單。

![](https://i.imgur.com/p2rUviG.jpg)

---
## 2. 設置Zabbix Trigger表達式

由於我們需要持續觀察`Computer`節點上面CPU狀態，Trigger設定這邊可以直接透過語法進行觀察如下:

* 需要連續三分鐘`CPU使用率平均值超過80%`觸發報警

    * 名稱（Name）
        * 可以輸入: "CPU load too high on 'node-1.domain.tld' for 3 minutes"作為值。這個值會作為觸發器的名稱被現實在列表和其他地方。
    * 表達式（Expression）
```bash=
{node-1.domain.tld:system.cpu.util[,idle].max(3m)}<20
```
> Trigger 觀察主要透過`system.cpu.util[,idle]`狀態顯示，反向思考如果CPU能使用空閒小於20%，即CPU佔用超過80%立即觸發報警

* 把Trigger表達式打在`Expression`中

![](https://i.imgur.com/pI0Ztrh.jpg)
> 可以限制Trigger Severity警告提式: 這邊直接設定為`High`警告

完成後，點擊添加（Add）。新的觸發器將會顯示在觸發器列表中。

---
## 3. 顯示觸發器狀態

* 創建好的Trigger即可馬上列表出目前狀態

![](https://i.imgur.com/L2o9mzF.jpg)

如果要查看Trigger目前監控狀態可以透過，前往監控（Monitoring） → 觸發器（Triggers），3分鐘後（我們需要等待3分鐘以評估這個觸發器的3分鐘平均值），觸發器會在這裡顯示。應該會有一個綠色的'OK'在'狀態（Status）'列中閃爍。

![](https://i.imgur.com/pgFiP79.jpg)

接下來可以透過存放於Compute `node-1.domain.tld`相關VM，來提升CPU使用率，達到Compute超標，這邊範例測試為`超標CPU 50%`表示為警告。

* 一般情況，綠色為`system.cpu.util[,idle]`狀態幾乎是很空閒的
![](https://i.imgur.com/wcVt11H.jpg)

* 超標狀態，可以明顯看到右側綠色能使用的`system.cpu.util[,idle]`明顯剩餘50%以下
 
![](https://i.imgur.com/f59EMx8.jpg)

從Zabbix 觀察Trigger狀態後，此處出現一個閃爍的紅色'PROBLEM',顯然，這說明了CPU負載已經超過了你在觸發器裡定義的閾值。
![](https://i.imgur.com/y8zNGRQ.jpg)

---

## 4. 透過Python獲得Zabbix API 獲取Trigger超標狀態

* 參考 GitHub: [連結](https://github.com/yylin1/zabbix-get-api/tree/master/zabbix-API)

此部分主要透過API方式取得Trigger.get欄位超標狀態，如果出現Trigger顯示為紅色閃爍的紅色`PROBLEM`，即可抓取到`status:0`的狀態，如果沒有超標則無法獲得資訊。

* 執行狀態可以直接獲得超標狀態
```bash=
$ python zabbix-get-trigger.py 

Trigger message list:  [ CPU load too high on 'node-1.domain.tld' for 3 minutes limit 50% ] view :status 0
```
    
* 若無超標及無法抓到`Trigger`列表的狀態
```bash=
$ python zabbix-get-trigger.py 

No Trigger load high problem in list.
```
---

### 參考資料:
* [[原创]Python利用Zabbix API定时报告存在报警的机器（更新：针对zabbix3.x）](https://tech.cuixiangbin.com/?p=757)
* [Zabbix Trigger表達式實例](http://pengyao.org/zabbix-trigger-example-1.html)
