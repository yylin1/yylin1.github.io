---
title: 如何自行編譯 HPL-GPU 來測試 Benchmark
toc: true 
date: 2018-10-23 21:11:49 
categories: 
- Linkpack
tags: 
- linpack
- GPU
- benchmark
thumbnail: /images/hpl_gpu.png
---
構建 NVIDIA CUDA Linpack 環境執行環境非常困難，網路上訊息非常少，然後linkpack測試更新版本已經有一段時間，記錄實作主要參考「[Hybrid HPL(GPU版HPL)安装教程](http://www.powerxing.com/hybrid-hpl/)」與「[AWS-GPUとスパコンを比較する方法-スパコン用ベンチマークソフトを動かしてみる](https://qiita.com/shibacow/items/aa83cfc81ed493a4c25d)」兩個文章教學，並嘗試運行現在環境支援的版本，部署過程記錄。

<!-- more -->

## 環境部署資訊
#### Linpack 部署的版本資訊：
* Mpich: v3.2.1
* Openmpi: v1.10.3
* Intel MKL: l_mkl_2019.0.117
* Linpack: hpl-2.0_FERMI_v15

## 實驗環境
安裝作業系統採用 Ubuntu 16.04 Desktop，測試環境為實體機器：

| Role | vCPU | RAM |Extra Device|
|-------- | -------- | -------- | -------- |
|ubuntu | 8 |	16G	|GTX 1060 6G|


## 事前準備
- 測試 Linkpack 之前，需要確保以下條件達成： 
確認環境是否安裝以下 `NVIDIA driver`、`CUDA`、`Intel MKL``Openmpi`、`mpich2`，並設定好環境變數。



### 安裝 NVIDIA驅動和CUDA Tookit
由於 CUDA Toolkit中，安裝時已包含了NVIDIA Driver，可一併安裝

```bash=
$ wget http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/cuda-repo-ubuntu1604_9.1.85-1_amd64.deb
$ sudo dpkg -i cuda-repo-ubuntu1604_9.1.85-1_amd64.deb
$ sudo apt-key adv --fetch-keys http://developer.download.nvidia.com/compute/cuda/repos/ubuntu1604/x86_64/7fa2af80.pub
$ sudo apt-get update && sudo apt-get install -y cuda
```

測試 NVIDIA Dirver 與 CUDA 是否有安裝完成：

```bash=
$  lsmod | grep nvidia
nvidia_uvm            790528  0
nvidia_drm             40960  2
nvidia_modeset       1089536  3 nvidia_drm
drm_kms_helper        167936  1 nvidia_drm
drm                   360448  5 nvidia_drm,drm_kms_helper
nvidia              14032896  96 nvidia_modeset,nvidia_uvm
ipmi_msghandler        45056  2 nvidia,ipmi_devintf

$ cat /usr/local/cuda/version.txt
CUDA Version 9.2.148

$ nvidia-smi
Tue Oct  2 18:15:47 2018
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 396.44                 Driver Version: 396.44                    |
|-------------------------------+----------------------+----------------------+
| GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================|
|   0  GeForce GTX 106...  Off  | 00000000:03:00.0  On |                  N/A |
| 39%   31C    P8     7W / 120W |     52MiB /  6077MiB |      0%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0      1603      G   /usr/lib/xorg/Xorg                            49MiB |
+-----------------------------------------------------------------------------+
```

---

### 準備 Linpack

Link : https://developer.nvidia.com/rdp/assets/cuda-accelerated-linpack-linux64
從上面的連結，登入CUDA註冊開發者會員，下載linpack for Linux64版本，這裡下載到的版本為`hpl-2.0_FERMI_v15.tgz`。

> 參考連結
> [Hybrid HPL(GPU版HPL)安装教程](http://www.powerxing.com/hybrid-hpl/)
> [AWS-GPUとスパコンを比較する方法-スパコン用ベンチマークソフトを動かしてみる](https://qiita.com/shibacow/items/aa83cfc81ed493a4c25d)

--- 
### 安裝INTEL MKL

透過連結`註冊帳號`
https://software.intel.com/en-us/qualify-for-free-software

![](https://i.imgur.com/9KxitwB.png)
註冊後，它會向您發送序列號於信箱，以便進行安裝準備。

這邊是下載最新[l_mkl_2019.0.117.tgz](http://registrationcenter-download.intel.com/akdlm/irc_nas/tec/13575/l_mkl_2019.0.117.tgz)版本

下載取得`l_mkl_2019.0.117.tgz`後，即可透過`install.sh`運行安裝。

```bash=
$ tar zxvf l_mkl_2019.0.117.tgz
$ cd l_mkl_2019.0.117
```

Intel mkl的安裝很簡單的，每一步也都有說明，按`Enter`繼續下一步預設設定安裝即可，安裝到某一步會要求輸入序列號，申請30天試用版所給的那個序列號。
```bash=
$ sh ./install.sh

--------------------------------------------------------------------------------
Initializing, please wait...
--------------------------------------------------------------------------------
Welcome
--------------------------------------------------------------------------------
Welcome to the Intel(R) Math Kernel Library 2019 for Linux*
--------------------------------------------------------------------------------


You will complete the following steps:
   1.  Welcome
   2.  License Agreement
   3.  Options
   4.  Installation
   5.  Complete

--------------------------------------------------------------------------------

--------------------------------------------------------------------------------
Press "Enter" key to continue or "q" to quit:
License Agreement
--------------------------------------------------------------------------------

```
確認後會安裝一些套件，這裡就可以看到MKL預設情況下，會安裝在`/opt/intel`下面。
```bash=+
------------------------
Options > Pre-install Summary
--------------------------------------------------------------------------------
Install location:
    /opt/intel

Component(s) selected:
    Intel Math Kernel Library 2019 for C/C++                               2.6GB
        Intel MKL core libraries for C/C++
        Intel TBB threading support
        GNU* C/C++ compiler support

    Intel Math Kernel Library 2019 for Fortran                             2.6GB
        Intel MKL core libraries for Fortran
        GNU* Fortran compiler support
        Fortran 95 interfaces for BLAS and LAPACK

   Install space required:  2.8GB
```

編譯完成後，即會顯示安裝資訊。

```bash=+
------------------------
Complete
--------------------------------------------------------------------------------
Thank you for installing Intel(R) Math Kernel Library 2019 for Linux*.

If you have not done so already, please register your product with Intel
Registration Center to create your support account and take full advantage of
your product purchase.

Your support account gives you access to free product updates and upgrades
as well as Priority Customer support at the Online Service Center
https://supporttickets.intel.com.
```
> 完整安裝過程於[Gist](https://gist.github.com/yylin1/6b30bfb9456f74ae9769e1d65ee60bb2)。
---

### 安装mpich2
```bash=
$ wget http://www.mpich.org/static/downloads/3.2.1/mpich-3.2.1.tar.gz
tar zxvf mpich-3.2.1.tar.gz
$ cd mpich-3.2.1
./configure -prefix=/home/username/mpich
$ make
$ make install
```

#### 配置環境
打開/etc/environment 
```bash=
$ vim /etc/environment 
```
將自己的路徑添加到PATH最後，注意別忘了冒號“：”，添加後的PATH如下
```bash=
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/local/cuda-9.2/bin:/home/username/mpich/bin"
```
保存退出，在終端輸入`source /etc/environment`
再輸入`echo $PATH`測試發現已經更新，環境變量配置成功。

---------------------

本文来自 ForTheDreamSMS 的CSDN 博客 ，全文地址请点击：https://blog.csdn.net/baidu_34045013/article/details/78237842?utm_source=copy 


> 安裝參考:[ubuntu16.04安裝配置mpich2](https://blog.csdn.net/baidu_34045013/article/details/78237842)

### 安裝openmpi

```bash=
$ wget -c https://www.open-mpi.org/software/ompi/v1.10/downloads/openmpi-1.10.3.tar.gz

$ tar zxvf openmpi-1.10.3.tar.gz
$ cd openmpi-1.10.3
$ ./configure --prefix=/opt/openmpi
$ make
$ sudo make install
```
安裝`make`和`make instal`需要一段時間，等待完成即可，openmpi環境配置會在後面統一設定。

> 參考 : [OpenMPI設定叢集環境](https://codertw.com/%E7%A8%8B%E5%BC%8F%E8%AA%9E%E8%A8%80/543992/)

---
## 配置環境變量

首先更改環境變量PATH：

```bash=
sudo vim /etc/environment
```
在PATH變量加上/usr/local/cuda-9.2/bin,前面要有分號，後面沒有，修改後例如下面這樣：
```bash=
PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/usr/local/cuda-9.2/bin:/home/username/mpich/bin"
```
保存文件，然後再執行：
`source /etc/environment`
完成後，可以執行echo $PATH,查看是否修改成功

接著還需更改`ldconfig`
```bash=
cd /etc/ld.so.conf.d/
sudo vim hpl.conf
```
輸入如下內容
```bash=
/usr/local/cuda-9.2/lib64
/lib
/opt/intel/mkl/lib/intel64
/opt/intel/lib/intel64
/home/ubuntu/hpl/src/cuda
```
最後一行`/home/使用者/hpl/src/cuda`是編譯HPL時才需要改的，在這裡一併修改。這個目錄就是編譯hpl時，hpl的路徑。

添加上述內容並保存後，執行
```bash=
sudu ldconfig
```
可以輸入下面命令進行檢驗，有輸出內容就對了
```
sudo ldconfig -v | grep cuda
```
接著還要執行Intel MKL的環境變量設置腳本
```bash=
export LD_LIBRARY_PATH=/opt/intel/mkl/lib/intel64:/opt/intel/compilers_and_libraries/linux/lib/intel64:/home/ubuntu/hpl/src/cuda:/opt/openmpi/lib
export PATH=/opt/openmpi/bin:$PATH

source /opt/intel/compilers_and_libraries_2019.0.117/linux/mkl/bin/mklvars.sh intel64
```
請確認以上路徑與當前環境上所有套件的路徑是否對應存在，再執行
```
source ~/.bashrc
```
這樣，環境變量就設置好了。最好`echo $PATH`查看下是否多了一行intel的信息，如果沒有配置成功的話，在編譯HPL時會提示/usr/bin/ld: cannot find -liomp5的錯誤。


## 開始編譯Linpack benchmark for CUDA

這邊將`hpl-2.0_FERMI_v15.tgz`解壓縮放置主目錄下`hpl`文件夾，可以依照自己設定的路徑對應編譯。

```bash=
$ tar -xvf hpl-2.0_FERMI_v15.tgz –C ~/hpl
$ cd ~/hpl

$ ls
bin  BUGS  COPYRIGHT  CUDA_LINPACK_README.txt  HISTORY  include  INSTALL  lib  Make.CUDA  Makefile  makes  Make.top  man  README  setup  src  testing  TODO  TUNING  www
```

### 編譯Make.CUDA編輯配置
這時還需要編輯`Make.CUDA`測試環境參考[連結](https://gist.github.com/yylin1/55fbdb799064837b45478c8b165f5c82#file-make-cuda)，需更改Make.CUDA中的TOPdir為hpl的目錄。

```bash=
103  TOPdir = /home/ubuntu/hpl

132 LAdir        = /opt/intel/mkl/lib/intel64
133 LAMP5dir     = /opt/intel/compilers_and_libraries/linux/lib/intel64

134 LAinc        = -I/opt/intel/mkl/include
```

接著可以開始編譯了
```bash=
cd ~/hpl
make arch=CUDA
```
> 如果沒有提示錯誤，就是編譯成功了。

編譯完成後，還需要修改`~/hpl/bin/CUDA/run_linpack`中的[HPL_DIR](https://gist.github.com/yylin1/55fbdb799064837b45478c8b165f5c82#file-run_linpack)為你hpl的路徑

```bash=
HPL_DIR=/home/ubuntu/hpl
```
修改完成後就可以開始測試了。

測試之前建議把HPL.dat的參數改小一點，N改成8000，這樣所需的測試時間少。也先把P，Q，PxQ都改成1，保證可以執行測試:

```bash=
$ mpirun -n 1 ./run_linpack
```

### 輸出結果
```bash=
$ mpirun -n 1 ./run_linpack
================================================================================
HPLinpack 2.0  --  High-Performance Linpack benchmark  --   September 10, 2008
Written by A. Petitet and R. Clint Whaley,  Innovative Computing Laboratory, UTK
Modified by Piotr Luszczek, Innovative Computing Laboratory, UTK
Modified by Julien Langou, University of Colorado Denver
================================================================================

An explanation of the input/output parameters follows:
T/V    : Wall time / encoded variant.
N      : The order of the coefficient matrix A.
NB     : The partitioning blocking factor.
P      : The number of process rows.
Q      : The number of process columns.
Time   : Time in seconds to solve the linear system.
Gflops : Rate of execution for solving the linear system.

The following parameter values will be used:

N      :   25000    30000
NB     :     768     1024     1280     1536
PMAP   : Row-major process mapping
P      :       1
Q      :       1
PFACT  :    Left
NBMIN  :       2
NDIV   :       2
RFACT  :    Left
BCAST  :   1ring
DEPTH  :       1
SWAP   : Spread-roll (long)
L1     : no-transposed form
U      : no-transposed form
EQUIL  : yes
ALIGN  : 8 double precision words

--------------------------------------------------------------------------------

- The matrix A is randomly generated for each test.
- The following scaled residual check will be computed:
      ||Ax-b||_oo / ( eps * ( || x ||_oo * || A ||_oo + || b ||_oo ) * N )
- The relative machine precision (eps) is taken to be               1.110223e-16
- Computational tests pass if scaled residuals are less than                16.0

================================================================================
T/V                N    NB     P     Q               Time                 Gflops
--------------------------------------------------------------------------------
WR10L2L2       25000   768     1     1              43.07              2.419e+02
--------------------------------------------------------------------------------
||Ax-b||_oo/(eps*(||A||_oo*||x||_oo+||b||_oo)*N)=        0.0040802 ...... PASSED
================================================================================
```

> 完整環境配置與設置有放到[Gits](https://gist.github.com/yylin1/55fbdb799064837b45478c8b165f5c82#file-make-cuda-L103)，可以提供參考。


#### #補充-直接使用 Docker測試HPL GPU: [參考連結](https://qiita.com/shibacow/items/459949bb3bc81a58526a)
