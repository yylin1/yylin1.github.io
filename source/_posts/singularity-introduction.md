---
title: Singularity 基礎介紹
tags:
  - Singularity
  - Container
thumbnail: /images/singularity.jpg
abbrlink: 51813
date: 2020-06-12 01:03:14
---

Singularity 是 `Greg Kurtzer` 在 Lawrence Berkley National 實驗室時，他特別針對大規模、跨節點 HPC 應用和 Deep Leaning 工作負載所開發出的容器化技術，而現在是由他的公司 [Sylabs](https://sylabs.io/) 持續對於可用性與安全性做更進一步的開發，也推出針對企業級版本 [Singulairty PRO](https://sylabs.io/singularity-pro/order-now)。Singularity 與其他標準容器格式(OCI)兼容，部署容易並能與所有現有應用和 HPC 基礎架構結合使用，包括資源管理、並行檔案系統、MPI、GPU、InfiniBand 和 FPGA 等。

> 與當前相容於 OCI Runtime 的解決方案相比，Singularity 有許多優勢，其中使用校驗和加密簽名進行驗證，來保證對所包含容器中應用和數據的信任，以及協助在非安全環境和雲端運行時環境的可重複性和有效性。

<!-- more -->

![](https://i.imgur.com/D6aZUVT.png)

## Singularity 基本概念

Singularity 主要具備輕量資源開銷小、啟動迅速、快速部署、方便遷移與擴展等等，且支援直接從 Docker Image 轉換為 Singularity SIF  Image。

使用 Singularity 有以下幾項好處：

- 限制容器使用者權限： Singularity 同時支援 root 使用者和非 root 使用者啟動，且容器啟動前後，使用者上情境保持不變，這使得使用者權限在容器內部和外部都是相同的。

> 這部分對於開發者會有部署困擾，非 root 使用者無法自行建構 Contaienr Image，需要額外通過利用使用者命名空間的 UID/GID 映射，允許非特權使用者以 `--fakeroot` 使用者的身份運行 Singularity Contaienr，等於使用者擁有與 root 幾乎相同的管理權限。 [[更多資訊: Fakeroot feature ]](https://sylabs.io/guides/3.5/user-guide/fakeroot.html#fakeroot-feature)

- 弱化完全隔離性： Singularity 強調容器服務的便捷性、可移動性和可擴展性，使用單個文件為容器 Singularity Image Format，而弱化容器程序的高度隔離性，性能損失更小、容器體積更小。
    - 開發時進入容器後常會與當前執行環境搞混，需自行修改配置配置來區分正在執行的是容器內還是容器外


- 單文件格式增強環境遷徙： Singularity 所依賴的東西都在 [Singularity Image Format ](https://github.com/hpcng/sif)(SIF™)文件中，可以輕鬆創建儲存在單文件中，並完整封裝容器環境，不需要再單獨打包/導入，直接複製整個 Image 文件，就可以將整個容器環境從當前系統轉移至其他系統環境。沒有複雜的緩存機制，並且提供特殊壓縮方式，建構出來的 Image 只需佔用非常少的硬碟空間，容器啟動成本應更低。
- This addresses the "complex storage" issue with Docker.

![](https://i.imgur.com/p7eXFKL.png)


- 和當前系統無縫銜接：容器內系統使用者權限、網路、UTS namespace 直接繼承當前主機配置，並且無需進入某個 Image 後再執行指令，可以直接在外部啟用交互模式執行 Image 並進行容器指令。
    

- 可以自行啟動無需依賴其他 Daemon： Singularity 提供的完全是一個運行時的環境，不使用時不需要運行單獨的 process，不佔用任何資源，因此所有不由 daemon 代為執行指令，因此解決容器取用資源限制和使用者權限安全問題。

- 限制執行權限優勢： 大多數執行容器都是對於受信任的容器使用者前提下運行容器，但 Singularity 允許不受信任的使用者，以可信任的方式執行不受信任的容器，防止使用者在容器升級執行權限的能力，方法是保持父行程和容器內的子行程之間連結，使這些子行程與當前使用者有相同權限，來傳遞執令。

## Reference
- http://blog.zxh.site/2018/05/15/Singularity/
- https://sylabs.io/guides/3.5/user-guide/security.html
- https://github.com/schebro/sif
- https://medium.com/sylabs/singularity-3-5-0-d6b21998f9dc
- https://singularity-tutorial.github.io/
- https://www.dazhuanlan.com/2019/10/28/5db5dd8ae521e/
- [Oveview of VM vs Docker vs Singularity](https://tin6150.github.io/psg/blogger_container_hpc.html)
- https://cyberlearn.hes-so.ch/pluginfile.php/2908762/mod_resource/content/1/singularity_hesso2019.pdf
