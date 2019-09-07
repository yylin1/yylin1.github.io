---
title: Parameter Server 學習筆記
toc: true
date: 2019-01-09 23:14:08
categories: 
- Note
tags: 
- distributed
- parmeter server
thumbnail: /images/PS-concept.png
---

本篇文章主要介紹分散式概念，對於`參數伺服器(Parameter Server)`架構釐清，並與主流常見 機器學習(Machine Learning) 框架使用分散式運算實際應用場景探討。

<!-- more -->

## 概念：

在過去十年中深度學習技術已成功應用於許多領域，例如語音辨識、影像分析、物件偵測等。

大規模數據上跑機器學習任務是過去至今，資料中心系統架構面臨的主要挑戰之一，要如何有效解決運算需求成長，能否承載大型神經網絡與巨量數據資料資源請求挑戰：

> 1. 要訪問這些巨量的參數超過單個機器容納的頻寬能力(Bandwidth)支持
> 2. 很多ML算法是序列性的，同步過程會影響性能損失
> 3. 在分佈式中，容錯能力是非常重要的。很多情況下，算法都是部署到雲環境中的（這種環境下，機器是不可靠的，並且job也是有可能被搶占的）；

為了解決這些問題，大神們提出了一種新的架構- Parameter Server(簡稱PS)。

--- 

## Paramter Server架構設計:

## 1. Paramter Server 整體架構

PS架構主要包括兩大部分，一個`參數伺服器群（Parameter Server group）`和`多個工作(Worker)群`:

* 在PS中，每個`Parameter`實際上都只負責分到的**部分參數**（Servers共同維持一個全局的共享參數），而每個Worker 也只分到部分數據和處理任務
* 每個子節點都只維護自己分配到的參數，自己部分更新之後，將計算結果（例如：梯度）傳回到主節點，進行全局的更新（比如All reduce平均操作之類的），主節點再向子節點傳送新的參數；

> Parameter Server 架構圖:

![](https://i.imgur.com/93RXOgT.png)

### 架構概念解釋：

- Server節點可以跟其他Server節點溝通，每個Server負責自己分到的參數，`Server group共同維持所有參數的更新`。

- Server manager node 負責維護一些`元數據(metadata)`的一致性，比如各個節點的狀態，參數的分配情況等。

- Worker節點之間無交流，只能跟自己對應的Server進行溝通。
- 每個Worker group有一個 task scheduler，負責向 Worker 分配任務，並且監控 Worker的運行情況。當有新的Worker加入或者退出，`task scheduler負責重新分配任務`。

- training data 會被 split 多個部分，一個 Worker 在本地將一部分訓練數據存儲在本地統計數據中。

> 範例: Parameter Server 示意圖  

![](https://i.imgur.com/HwYPB7c.gif)

上圖中，每個子節點都只維護自己分配到的參數（圖中的黑色），自己部分更新之後，將計算結果（例如：梯度）傳回到主節點，進行全局的更新（比如平均操作之類的），主節點再向子節點傳送新的參數。

> Ref: [[Parameter Server 详解-仙道菜]](https://blog.csdn.net/cyh_24/article/details/50545780)
---
## 2. Parameter Server 架構優勢

#### 1. 高效溝通(Efficient communication)

- 非同步(Asynchronous)訓鍊，使得計算不會被拖累，不需要停下來等一些機器(Worker)執行完一個iteration，這大幅減少延遲時間。

#### 2. 彈性一致性模型(Flexible consistency models)

- 允許用戶自定義一致性: 比如Sequential（序列式的，即完全同步）、Eventual（完全不同步的）和Bounded Delay（有條件的限制，可以允許用戶在限制的次數內異步，比如限制為3 次，如果某個節點已經超前了其他節點四次迭代了，那麼要停下等待同步。在整個訓練的過程中，Delay 可能是動態的，即 delay 的參數在訓練過程中可以變大或變小）。

#### 3. 擴展性強 (Elastic Scalability)

- 使用了一個分佈式 hash 表使得新的server 節點可以隨時動態的插入到集合中；因此，新增一個節點不需要重新運行系統。

#### 4. 錯誤容忍 (Fault Tolerance and Durability)

- 我們都知道，節點故障是不可避免的，特別是在大規模商用伺服器集群中。從非災難性機器故障中恢復，只需要1秒，而且不需要中斷計算。Vector clocks保證了經歷故障之後還是能運行良好。

#### 5. 易用性 (Ease of Use)

- 全局共享的參數可以被表示成各種形式：vector、matrices或者相應的 sparse 類型，這方便機器學習算法的開發。並且提供的線性代數數據類型都具有高性能的多線程庫。

--- 

## 3. Parameter Server 關鍵概念
### 1. (key, value)，Range Push and Pull

![](https://i.imgur.com/EvxBA4m.png)

- Parameter Server中，參數都是可以被表示成`(key, value)`的集合，比如一個最小化損失函數的問題，key 就是feature ID，而value 就是它的權值。對於稀疏參數，不存在的key，就可以認為是0。

- 把參數表示成kv (key, value)， 行式更自然、易於理解、更易於編程解；

- Workers 跟 Servers 之間通過 push 跟 pull 來溝通 : 
    - Worker 通過 **push** 將計算好的梯度發送到 Server，然後通過 **pull** 從Server 更新參數。
    - 為了提高計算性能和頻寬效率，Parameter Server 允許用戶使用 **Range Push** 跟 **Range Pull** 操作（使用區間更新的方式，那麼可以進行如下操作：
    
> w.push(R, dest)
> w.pull(R, dest)

發送和接送特定 Range中的 w 。

### 2. Key-value vectors

- 賦予每個 key 所對應的 value 一個向量概念或矩陣概念。

### 3. User-Defined Functions on the Server

- 伺服器端更新參數的時候還有計算正則項，這樣的操作可以由使用者自定義。

### 4. Asychronous Tasks and Dependency

同步(Synchronous)與非同步(Asynchronous)差異：

![](https://i.imgur.com/uBiLLcf.png)

如上圖，如果 iter1 需要在 iter0 computation，進行 push 跟 pull 都完成後才能開始;`Synchronous` 將所有計算完的梯度放在一起處理，當每次更新梯度時，需要等所以分發的資料計算完成，並回傳結果來把梯度累加計算平均，在進行更新變數。

> 好處在於使用 loss 的下降時比較穩定，壞處就是要等最慢的Work計算完成時間。

#### 補充Asychronous 架構說明：

![](https://i.imgur.com/3bZhCC9.png)
> Ref: [Paper][Online job scheduling in Distributed Machine Learning Clusters]
>> Fig.2: Asynchronous Training Workflow

- 系統中非訓練工作流程，作業中不同 Worker 的訓練進度不同步，並且每次參數伺服器在接收到來自 Worker 的梯度時，每次更新其參數。

- 在上面Fig.2 圖中，參數伺服器使用「new weight = old weight − stepsize × gradient」然後將更新的權重發送回 Worker，處理完整個數據塊後，Worker 將繼續訓練分配給它的下一個數據塊。

> Asychronous 的優點是能夠提高系統的使用效率（節省任務等待的過程），但是它的缺點就是容易降低算法的收斂速率。

---
考慮到使用者使用的時候會有不同的情况，Parameter Server 為使用者提供了多種任務依賴的方式：![](https://i.imgur.com/4Y4rov1.png)

> **Sequential**： 这里其实是 synchronous task，任务之间是有顺序的，只有上一个任务完成，才能开始下一个任务；
> **Eventual**： 跟 sequential 相反，所有任务之间没有顺序，各自独立完成自己的任务，
> **Bounded Delay**： 这是sequential 跟 eventual 之间的trade-off，可以设置一个 τ 作为最大的延时时间。也就是说，只有 >τ 之前的任务都被完成了，才能开始一个新的任务；极端的情况：
> 1. τ=0，情况就是 Sequential；
> 2. τ=∞，情况就是 Eventual；

---

### 5. User-Defined Filters（使用者自定義過濾）

在Worker節點這一端對梯度進行過濾，如果梯度並不是影響那麼大，就不佔用網絡去更新，等累積一段時間之後再去做更新。

對於機器學習優化問題比如梯度下降來說，並不是每次計算的梯度對於最終優化都是有價值的，使用者可以通過自定義的規則過濾一些不必要的傳遞，而再進一步壓縮頻寬花費：

- 發送很小的梯度值是低效率的：
    - 因此可以自定義設置，只`在梯度值較大的時候發送`

- 更新接近最優情況的值是低效的：
    - 所以只在`非最優的情況下發送`，可通過KKT來判斷
---
## 4. Paramter Server架構實現


### Vector Clock

- 為參數伺服器中的每個參數添加一個時間戳記，來跟蹤參數的更新和防止重複發送數據。基於此，溝通中的梯度更新數據中也應該有時間戳，防止重複更新。

- 如果每個參數都有一個時間戳，那麼參數眾多，時間戳也眾多。幸好 Parameter server在 push 跟 pull 的時候，都是 rang-based，這就帶來了一個好處：這個range裡面的參數共享的是同一個時間戳，這顯然可以大大降低了空間複雜度。

### Messages

- Message是節點間交互的主要格式。一條 message 包括：時間戳，len(range)對kv. $[vc(R),(k1,v1),...,(kp,vp)]kj∈Randj∈{1,...p}$

- 這是parameter server 中最基本的溝通格式，不僅僅是共享參數才有，task 的 message也是這樣的格式，只要把這裡的(key, value) 改成(task ID, 參數/返回值)。

> 由於機器學習問題通常都需要很高的網絡頻寬，因此信息的壓縮是必須的。


- key的壓縮：因為訓練數據通常在分配之後都不會發生改變，因此worker沒有必要每次都發送相同的key，只需要接收方在第一次接收的時候緩存起來就行了。第二次，worker不再需要同時發送key和value，只需要發送value和key list的hash就行。這樣瞬間減少了一般的通信量。

- value的壓縮：假設參數時稀疏的，那麼就會有大量的0存在。因此，為了進一步壓縮，我們只需要發送非0值。parameter server使用Snappy快速壓縮庫來壓縮數據、高效去除0值。

- key的壓縮和value的壓縮可以同時進行。


### Replication and Consistency

參數伺服器集群中每個節點都負責不同區域的參數，那麼，類似於hash table，使用hash ring進行實現，key和server id都插入到hash ring上。

![](https://i.imgur.com/Kpyx4DF.png)


### 備份和一致性

使用類似hadoop 的chain 備份方式，對於一個master 節點，如果有更新，先更新它，然後再去更新備份的伺服器。
![](https://i.imgur.com/a8R7kHL.png)
在更新的時候，由於機器學習算法的特點，可以將多次梯度聚合之後再去更新備份伺服器，從而減少帶寬。


## Server Management

由於key 的range 特性，當參數伺服器集群中增加一個節點時，步驟如下：

- Server Manager 節點給新節點分配一個key range，這可能會導致其他節點上的key range 切分。
- 新節點從其他節點上將屬於它的key range 數據取過來，然後也將slave 信息取過來。
- Server Manager廣播節點變動，其他節點得知消息後將不屬於自己key range 的數據刪掉
- 在第二步，從其他節點上取數據的時候，其他節點上的操作也分為兩步，第一複製數據，這可能也會導致key range 的切分。第二是不再接受和這些數據有關的消息，而是進行轉發，轉發到新節點。

- 在第三步，收到廣播信息後，節點會刪除對應區間的數據，然後，掃描所有的和R有關發送出去的還沒收到回复的消息，當這些消息回复時，轉發到新節點。

> 節點的離開與節點的加入是類似


### Worker Management
添加工作節點比添加參數伺服器節點要簡單一些，步驟如下：

- task scheduler給新節點分配一些數據
- 節點從網絡文件系統中載入數據，然後從伺服器端拉取參數
- task scheduler廣播變化，其他節點 free掉一些訓練數據

當一個節點離開的時候，task scheduler可能會尋找一個替代，但恢復節點是十分耗時的工作，同時，損失一些數據對最後的結果可能影響並不是很大。所以，系統會讓用戶進行選擇，是恢復節點還是不做處理。這種機制甚至可以允許用戶刪掉跑的最慢的節點來提升速度。




## 5. 何時使用分散式深度學習?


## ![](https://i.imgur.com/vTUS2wt.png)
> 分散式的深度學習並不總是最佳的選擇，需要視情況而定：

進行分散式訓練，由於同步、資料和參數與網路傳輸等，分散式系統相比單機訓練要多不少額外的必要開銷。

可能導致網路模型的訓練時間過長：
- 神經網路太大
- 資料量太大

> 上圖其參考選擇任務合適對應訓練環境場景，其中都要考慮到 「網路傳輸」與「總計算量」

--- 
# 總結

- 一般在資料量小，且各節點計算能力平均下，適合使用同步模式; 反之在資料量大與各節點效能差異不同時，適合用非同步。
- 系統性能跟算法收斂速率之間是存在一個平衡（trade-off） 的情況，你需要同時考慮
    - 算法對於參數非一致性的敏感度
    - 訓練數據特徵之間的關聯度
    - 節點硬碟存儲容量

- 什麼時候DL訓練任務，真的需要使用分散式深度學習



---

## Further Readings
- [[Distributed System] Intro-Distributed-Deep-Learning
](https://xiandong79.github.io/Intro-Distributed-Deep-Learning)
## References
- [[Distributed System] 小沙文的博客 - Parameter Server 学习](http://pengshuang.space/2017/07/23/Parameter-Server-%E5%AD%A6%E4%B9%A0/)
- [[[Distributed System] #周末识堂# 参数服务器 parameter server]](http://www.o-k.la/parameter-server.html)
- [[Distributed System]Parameter Server 详解](https://blog.csdn.net/cyh_24/article/details/50545780)
- [[[Distributed System] raincoffee - Parameter Server]](https://www.jianshu.com/p/d3e503ddd68e)
- [[Paper] Scaling Distributed Machine Learning with the Parameter Server](https://www.cs.cmu.edu/~muli/file/parameter_server_osdi14.pdf)
- [[Paper] Large Scale Distributed Deep Networks](http://www.cs.toronto.edu/~ranzato/publications/DistBeliefNIPS2012_withAppendix.pdf)
