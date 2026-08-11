# ch8. SUSE Linux Enterprise Server 15 - Virtualization & Containerization

虛擬化（Virtualization）與容器化（Containerization）都是在同一台實體主機上隔離工作負載的技術，但隔離層級不同：

| 類型       | 典型代表             | 隔離層級                                       | 意義                                   |
| ---------- | -------------------- | ---------------------------------------------- | -------------------------------------- |
| 系統虛擬化 | KVM + QEMU + libvirt | 虛擬整台機器（Guest OS）                       | 可跑不同核心／發行版，隔離強、開銷較大 |
| 容器化     | Docker 等            | 共用宿主核心，隔離行程／檔案系統／網路命名空間 | 啟動快、密度高，適合微服務與一致交付   |

本章說明 KVM／QEMU／libvirt 的角色與基本操作，以及 Docker 的安裝、權限與常用生命週期指令。

---

## 8-1. KVM、QEMU、libvirt

### 8-1-1. 演進與角色分工

現代 Linux 伺服器虛擬化常見「三層分工」：

| 元件        | 全名／定位                   | 角色                                                                                 |
| ----------- | ---------------------------- | ------------------------------------------------------------------------------------ |
| **KVM**     | Kernel-based Virtual Machine | 核心模組，把 Linux 變成 Hypervisor，提供硬體輔助虛擬化（Intel VT-x／AMD-V）          |
| **QEMU**    | Quick Emulator               | 使用者空間模擬器／VMM：模擬磁碟、網卡等裝置，並管理 VM 行程；搭配 KVM 時可近原生效能 |
| **libvirt** | 虛擬化管理層                 | 提供統一 API 與工具（`virsh`、`virt-manager`），協調儲存、網路、生命週期             |

#### 簡史脈絡

1. **QEMU 早期**: 以軟體模擬為主，可跨架構，但純模擬效能有限。
2. **硬體虛擬化普及**: CPU 提供 SVM／VMX 後，核心側需要高效 Hypervisor 介面。
3. **KVM 進入 Linux 核心**: 成為主流開源 Type-1／宿主核心整合路線，QEMU 負責裝置模擬並透過 `/dev/kvm` 加速。
4. **libvirt 出現**: 為了讓管理工具不必直接綁死某一 Hypervisor，提供 XML 網域定義、網路（NAT／bridge）、儲存池等抽象；同一套工具可管理 KVM、有時也延伸到其他後端。

- **意義**: 實務上說「用 KVM」多半是 **KVM（加速）+ QEMU（模擬）+ libvirt（管理）** 的組合，而不是單一程式。

```text
[virt-manager / virsh / virt-install]
              │
           libvirt
              │
     ┌────────┴────────┐
     │                 │
   QEMU 行程        虛擬網路／儲存
     │
   /dev/kvm（KVM）
     │
   硬體 CPU 虛擬化擴充
```

### 8-1-2. 安裝套件

```bash
sle15:~ # zypper in qemu-kvm libvirt virt-install
```

| 套件           | 意義                              |
| -------------- | --------------------------------- |
| `qemu-kvm`     | QEMU 與 KVM 加速相關元件          |
| `libvirt`      | 管理守护與函式庫（含 `libvirtd`） |
| `virt-install` | 以命令列建立／安裝 Guest 的工具   |

### 8-1-3. 確認硬體與模組

```bash
sle15:~ # grep -E 'svm|vmx' /proc/cpuinfo
sle15:~ # lsmod | grep kvm
```

| 檢查                | 意義                                        |
| ------------------- | ------------------------------------------- |
| `vmx`               | Intel VT-x 旗標出現在 cpuinfo               |
| `svm`               | AMD-V 旗標                                  |
| `lsmod \| grep kvm` | 確認 `kvm`、`kvm_intel` 或 `kvm_amd` 已載入 |

- 在 VMware 等巢狀虛擬化環境，需在外層 VM 啟用 **Virtualize Intel VT-x/EPT** 或同等選項，內層才看得到 SVM／VMX。
- 若 `grep` 無輸出，後續建立 KVM Guest 常會失敗或退回慢速模擬。

### 8-1-4. 啟動 libvirtd

```bash
sle15:~ # systemctl start libvirtd
sle15:~ # systemctl enable libvirtd
sle15:~ # systemctl status libvirtd
```

| 步驟     | 意義                            |
| -------- | ------------------------------- |
| `start`  | 立即啟動管理服務                |
| `enable` | 開機自動啟動                    |
| `status` | 確認 active，並查看錯誤日誌線索 |

- **意義**: `virsh`／`virt-manager` 多半透過 `libvirtd` 操作；服務未起會連不上 hypervisor。

### 8-1-5. 圖形管理：virt-manager

```bash
sle15:~ # zypper in virt-manager
sle15:~ # virt-manager
```

| 項目           | 意義                                                         |
| -------------- | ------------------------------------------------------------ |
| `virt-manager` | GUI 管理主控台，可建立 VM、調 CPU／RAM／磁碟／網卡、開主控台 |
| 適用           | 桌面或有 X11／Wayland 轉發的管理站                           |

- 無圖形環境時可改用 `virt-install` + `virsh`。

### 8-1-6. 命令列：virsh

```bash
sle15:~ # zypper in libvirt-client
sle15:~ # virsh
sle15:~ # virsh help
```

#### 連線 URI

```bash
sle15:~ # virsh [-c qemu:///system] uri
sle15:~ # virsh [-c qemu+ssh://<user>@<ip>/system] uri
```

| URI                         | 意義                                                |
| --------------------------- | --------------------------------------------------- |
| `qemu:///system`            | 本機系統層連線（管理系統範圍的 VM，通常需足夠權限） |
| `qemu:///session`           | 使用者 session 範圍（未在本節展開）                 |
| `qemu+ssh://user@ip/system` | 經 SSH 管理遠端 libvirt                             |

```bash
sle15:~ # virsh list --all
```

| 指令               | 意義                          |
| ------------------ | ----------------------------- |
| `virsh list`       | 列出**正在執行**的 domain     |
| `virsh list --all` | 含關機／其他狀態的全部 domain |

#### 生命週期

```bash
sle15:~ # virsh start <domain name> [--console]
sle15:~ # virsh destroy <domain name>
sle15:~ # virsh console <domain name>
```

| 指令        | 意義                                                   |
| ----------- | ------------------------------------------------------ |
| `start`     | 啟動已定義的 VM（domain）                              |
| `--console` | 啟動並嘗試接上序列主控台                               |
| `destroy`   | **強制電源中斷式**關閉（非 Guest 內優雅 shutdown）     |
| `console`   | 連到 Guest 序列主控台（需 Guest 有設定 getty／serial） |

- 優雅關機應優先在 Guest 內 `shutdown`，或使用 `virsh shutdown`（需 ACPI 回應）。
- `destroy` 適合卡住時救援，有檔案系統不一致風險。

### 8-1-7. 建議操作步驟（KVM）

1. 確認 CPU 虛擬化旗標與 `kvm` 模組
2. 安裝 `qemu-kvm`、`libvirt`、`virt-install`
3. `systemctl enable --now libvirtd`
4. 用 `virt-manager` 或 `virt-install` 建立第一台 Guest
5. `virsh list --all` 確認狀態
6. `virsh console` 或圖形主控台完成 Guest 安裝
7. 平日以 `shutdown`／`start` 管理；必要時才 `destroy`

### 學習心得：KVM／QEMU／libvirt

- 先確認硬體虛擬化，再談軟體堆疊，可避免「套件裝好卻極慢／無法啟動」。
- libvirt 的價值是統一管理面；學會 `virsh` 比只會點 GUI 更利於自動化。
- `destroy` ≠ 正常關機；文件與操作習慣要分開。

### 練習：KVM

1. 用一句話區分 KVM、QEMU、libvirt 的職責。
2. 寫出檢查 Intel／AMD 虛擬化是否啟用的指令。
3. 比較 `virsh shutdown` 與 `virsh destroy` 的風險差異。
4. 說明 `qemu:///system` 與 `qemu+ssh://user@host/system` 各適用何時。

---

## 8-2. Docker

### 8-2-1. 演進歷史（精要）

| 階段              | 說明                                                                                |
| ----------------- | ----------------------------------------------------------------------------------- |
| 早期 chroot／jail | 僅有限隔離，管理與搬運不易                                                          |
| LXC 等容器技術    | 利用 Linux namespaces、cgroups 做行程級隔離                                         |
| **Docker** 普及   | 以映像（Image）、容器（Container）、Dockerfile、Registry 降低「在我機器可跑」問題   |
| 生態擴展          | Compose、Swarm、與 Kubernetes 等編排系統銜接；執行時亦出現 containerd／CRI-O 等分工 |

- **意義**: Docker 常被當成「容器」代名詞，本質是把應用與其依賴打包成可移植映像，並用命名空間隔離行程，同時仍**共用宿主 Linux 核心**。

#### 與虛擬機對照

| 項目       | VM（KVM）                 | Container（Docker）      |
| ---------- | ------------------------- | ------------------------ |
| Guest 核心 | 有獨立核心                | 通常無，用 Host kernel   |
| 啟動速度   | 較慢                      | 快                       |
| 隔離強度   | 較強                      | 較弱（仍需補強安全政策） |
| 磁碟占用   | 較大                      | 層狀映像，較易共用       |
| 典型用途   | 完整 OS、異質核心、強隔離 | 應用交付、CI、微服務     |

### 8-2-2. 安裝與啟動

```bash
sle15:~ # zypper in docker
sle15:~ # systemctl enable docker --now
```

| 步驟           | 意義                                                                     |
| -------------- | ------------------------------------------------------------------------ |
| 安裝 `docker`  | 提供引擎與 CLI（實際套件名稱依 SLES 模組／註冊可能為 `docker` 相關套件） |
| `enable --now` | 開機自動啟動並立即啟動 **dockerd**                                       |

### 8-2-3. 使用者權限

```bash
sle15:~ # usermod -aG docker <user>
```

| 步驟                       | 意義                   |
| -------------------------- | ---------------------- |
| 將使用者加入 `docker` 群組 | 可免每次 `sudo docker` |
| 重新登入                   | 群組變更才生效         |

- **安全提醒**: 能對 Docker socket 下指令，通常等同近似 root 能力；正式環境應限制成員並搭配政策。

```bash
sle15:~ # docker info
```

- **意義**: 確認引擎版本、儲存驅動、是否正常連到 daemon。

### 8-2-4. 執行容器

```bash
sle15:~ # docker run -it centos /bin/bash
sle15:~ # docker run -itd --name myos alpine /bin/sh
```

| 選項                            | 意義                             |
| ------------------------------- | -------------------------------- |
| `-i`                            | interactive，保持 stdin 開啟     |
| `-t`                            | 配置偽終端（TTY）                |
| `-d`                            | detached，背景執行               |
| `--name`                        | 指定容器名稱，便於後續操作       |
| 映像名（如 `centos`／`alpine`） | 本地沒有時會嘗試自 Registry 拉取 |
| 最後的命令                      | 容器內 PID 1 要跑的行程          |

- `-it` 適合進入互動 shell。
- `-itd` 適合背景長駐；要用時再 `docker exec` 進入。

### 8-2-5. 在運行中容器執行命令

```bash
sle15:~ # docker exec <container_id> /bin/sh
sle15:~ # docker exec -it <container_id> /bin/sh
```

| 指令                 | 意義                                        |
| -------------------- | ------------------------------------------- |
| `docker exec`        | 在**已運行**容器中再起一個行程              |
| 與 `docker run` 差異 | `run` 建立並啟動新容器；`exec` 進入既有容器 |

### 8-2-6. 檢視與刪除

```bash
sle15:~ # docker ps
sle15:~ # docker ps -a
sle15:~ # docker ps -aq

sle15:~ # docker rm <container_id>
```

| 指令            | 意義                                |
| --------------- | ----------------------------------- |
| `docker ps`     | 顯示**執行中**容器                  |
| `docker ps -a`  | 含已停止                            |
| `docker ps -aq` | 只輸出 ID（利於腳本批次）           |
| `docker rm`     | 刪除容器（執行中需先停，或加 `-f`） |

- 原文若寫 `dock ps`，正確指令為 **`docker ps`**。

### 8-2-7. 啟動／停止／重啟／強制停止

```bash
sle15:~ # docker start <container_id>
sle15:~ # docker stop <container_id>
sle15:~ # docker restart <container_id>
sle15:~ # docker kill <container_id>
```

| 指令      | 意義                                |
| --------- | ----------------------------------- |
| `start`   | 啟動已存在但停止的容器              |
| `stop`    | 送停止訊號，給行程優雅結束時間      |
| `restart` | 停止後再啟動                        |
| `kill`    | 強制送 SIGKILL 類強制結束，較不優雅 |

### 8-2-8. 建議操作步驟（Docker）

1. 安裝並 `systemctl enable --now docker`
2. `docker info` 確認 daemon 正常
3. （可選）`usermod -aG docker` 後重新登入
4. `docker run -it alpine /bin/sh` 驗證拉取與互動
5. 練習 `ps`／`stop`／`start`／`rm` 生命週期
6. 背景容器用 `-d --name`，再以 `exec -it` 進入
7. 正式環境再學習映像建置（Dockerfile）、Volume、網路與編排

### 學習心得：Docker

- 容器不是小型 VM；核心與核心模組仍來自主機，驅動／系統呼叫相容性要納入評估。
- 先掌握 `run`／`ps`／`exec`／`stop`／`rm`，再進入編排，曲線較平滑。
- `docker` 群組權限等同高風險管理能力，不應隨便開放給所有登入者。

### 練習：Docker

1. 比較 `docker run` 與 `docker exec`。
2. 說明 `-it` 與 `-d` 組合分別適合什麼情境。
3. 寫出列出全部容器 ID 並理解其腳本用途的指令。
4. 解釋為何加入 `docker` 群組前要評估安全性。

---

## 8-3. 虛擬化與容器如何選擇

| 需求                               | 較適合                                    |
| ---------------------------------- | ----------------------------------------- |
| 需要完整 Guest OS、異核、強隔離    | KVM＋libvirt                              |
| 快速交付應用、CI、密度與可移植映像 | Docker 容器                               |
| 巢狀實驗（在 VM 裡再學容器）       | 外層 KVM／VMware，內層 Docker             |
| 統一生命週期管理 API               | libvirt（VM）／Compose／K8s（容器，進階） |

兩者可並存：例如在 KVM Guest 中跑 Docker，或用 libvirt 管測試機、用容器跑應用單元。

---

## 步驟總結

1. **虛擬化路線**: 檢查 SVM／VMX → 安裝 qemu-kvm／libvirt → 啟用 libvirtd → virt-manager 或 virsh 建立 VM
2. **日常 VM 操作**: `virsh list --all` → `start`／`shutdown` → 必要時 `console`／`destroy`
3. **容器路線**: 安裝 docker → 啟用服務 →（審慎）加入 docker 群組 → `docker info`
4. **容器生命週期**: `run` → `ps` → `exec` → `stop`／`start` → `rm`
5. **依負載選技術**: OS 級隔離選 KVM；應用級打包選 Docker

---

## 總結

本章建立 SLES 上的兩大現代部署能力：以 **KVM＋QEMU＋libvirt** 管理虛擬機，以 **Docker** 管理容器。前者強調硬體輔助虛擬化與完整機器抽象，後者強調共用核心下的快速打包與執行。學會先驗證虛擬化旗標與 daemon 狀態，再熟練 `virsh` 與 `docker` 的生命週期指令，即可在實驗與正式環境中選擇合適的隔離模型，並為後續網路、儲存與編排主題打下基礎。
