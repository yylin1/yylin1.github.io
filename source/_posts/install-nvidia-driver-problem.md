---
title: 安裝NVIDIA顯卡驅動時可能遇到的問題
toc: true
date: 2018-12-01 23:21:20
categories: 
- Driver
tags:
- ubuntu
- GPU
- NVIDIA Driver
thumbnail: /images/nvidia-driver-problem.png
---

本篇主要介紹如何解決 Ubuntu 環境安裝 NVIDIA GPU Driver 後，會出現「循環登入的問題」，當裝完驅動重啟後，輸入登錄密碼之後，桌面一閃就退回到登錄界面了，然後就陷入到了輸入密碼登錄、彈出的循環，這時就需要進行顯卡驅動程序的卸載重裝。
<!-- more -->

## 卸載方法如下:

首先在登錄介面進入到 Linux 的 shell ie tty model，同時按下 Ctrl+Alt+F1 （F1~F6其中一個就可以），然後輸入使用者輸入使用者帳號密碼，成功進入到 shell，開始卸載 NVIDIA 驅動：

卸載乾淨所有安裝過的 NVIDIA 驅動

```bash=
$ sudo apt-get remove --purge nvidia-*
$ sudo apt-get autoremove
```

並透過直接下驅動解除安裝

```bash=
$ sudo nvidia-uninstall
```

再次檢查，透過 dpkg 檢查是否有非透過 apt-get 安裝的需要移除

```bash=
$ sudo dpkg -l 'nvidia'
$ sudo dpkg --remove nvidia-{name}
```

完成步驟後重啟系統

```bash=
$ reboot 
```

### 重新安裝 NVIDIA Driver 
官網連結：[NVIDIA Driver 下載連結](https://www.nvidia.com.tw/Download/index.aspx?lang=tw)

> 這邊測試環境GPU 為 GTX 1080 

再次透過登入介面 `Ctrl+Alt+F1` 切換到 `tty1` 執行

```bash=
# 關閉X server
$ sudo service lightdm stop    
$ sudo  ./NVIDIA-Linux-x86_64-410.78.run -no-x-check -no-nouveau-check -no-opengl-files

# -no-x-check 安裝時關閉X Server ;
# -no-nouveau-check 安装驅動時禁用Nouveau
# -no-opengl-files 安装時只裝驅動檔案，不安裝Opengl
```

安裝好後即，再次重啟電腦 `reboot`

```bash=
sudo service lightdm restart
```

> 重啟後即可透過UI介面登入環境，就可以解決`循環登入的問題`。

測試 NVIDIA Dirver 與 CUDA 是否有安裝完成：

```bash=
$ sudo nvidia-smi
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 390.87                 Driver Version: 390.87                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 108...  Off  | 00000000:02:00.0  On |                  N/A |
| 26%   40C    P8    16W / 250W |    328MiB / 11173MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0       837      C   python                                       157MiB |
|    0      1250      G   /usr/lib/xorg/Xorg                           159MiB |
+-----------------------------------------------------------------------------+
```

### 補充：安裝NVIDIA顯卡驅動是可能遇到的問題：
出現 `An error occurred while p    erforming the step : "Building kernel modules" `這個問題 : 

#### 問題原因
- Linix系統的內核是在不斷更新的，而安裝的NVIDIA驅動是之前下載好的，沒有更新，因此安裝過程中無法創建內核。

#### 解決方法
- 這時候從[NVIDIA下載驅動](https://www.nvidia.cn/Download/index.aspx?lang=tw/)對應新版本的驅動，並安裝執行以上步驟即可。

---
> 相關狀況參考資料 | 
[[Script] Install CUDA Toolkit v9.0 and cuDNN v7.0 on Ubuntu 16.04](https://gist.github.com/ashokpant/5c4e9481615f54af4025ab2085f85869)
[ubuntu 16.04 循环登录](https://blog.csdn.net/TaylorMei/article/details/79133369) 
[Ubuntu 16.04 安裝 CUDA + cuDNN + nvidia driver 的踩雷心得 (非安裝步驟詳解)](https://medium.com/@afun/ubuntu-16-04-%E5%AE%89%E8%A3%9D-cuda-cudnn-nvidia-driver-%E7%9A%84%E8%B8%A9%E9%9B%B7%E5%BF%83%E5%BE%97-%E9%9D%9E%E5%AE%89%E8%A3%9D%E6%AD%A5%E9%A9%9F%E8%A9%B3%E8%A7%A3-b13121d95025)
[安裝 NVIDIA Docker 2 來讓容器使用 GPU](https://kairen.github.io/2018/02/17/container/docker-nvidia-install/)
