# ch1. 練習題 — 虛擬機安裝、分割、網路與 SSH

目標：選用一套桌面 Hypervisor，建立 **兩台** 最新 **SLES 15** 虛擬機，完成分割、網路互通、SSH 與主機名稱設定，並能以指令驗證結果。

**建議環境**

- Host：Windows / Linux / macOS（依所選 Hypervisor）
- Guest：SUSE Linux Enterprise Server 15（最新可用 SP）× 2
- 網路：先能上網下載／註冊（可選），並讓兩台 VM 互相連通

---

## 練習 0. 安裝 Hypervisor（三選一）

任選一套跨平台／桌面虛擬化軟體並完成安裝。

- [Desktop Hypervisor](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion)
- [Download VirtualBox](https://www.virtualbox.org/wiki/Downloads)
- [Download QEMU](https://www.qemu.org/download/)

---

## 練習 1. 安裝兩台最新 SLES 15 虛擬機

### 1-0. 建立 VM 與安裝

1. 下載官方提供的 **最新 SLES 15** 安裝媒體（ISO）。
2. 建立 **兩台** Guest（建議資源各至少：2 vCPU、2–4 GB RAM、磁碟 ≥ 40 GB）。
3. 兩台皆完成安裝並可登入（可用 Minimal／Text 角色以節省資源）。
4. 分割區規劃（Partition）

安裝時（或 Expert Partitioner）請明確規劃下列掛載點／用途，並記錄實際裝置名稱與大小：

| 掛載點／用途 | 最低要求（可自訂，但須存在）              | 建議記錄項目                        |
| ------------ | ----------------------------------------- | ----------------------------------- |
| `/boot`      | 獨立分割（若開機模式需要；UEFI 另需 ESP） | 裝置、檔案系統、大小                |
| `swap`       | 交換空間                                  | 大小                                |
| `/`          | 根檔案系統                                | 檔案系統類型（如 btrfs／ext4／xfs） |
| `/home`      | 獨立於根目錄                              | 檔案系統類型、大小                  |

> 若為 **BIOS + GPT**，可能另需 **BIOS Boot Partition**；若為 **UEFI**，需 **EFI System Partition（ESP）**。請依實際開機模式補上並說明。

---

## 練習 2. 網路設定與兩機互通

兩台 VM 分別稱為 **VM-A**、**VM-B**（可自訂 hostname，見練習 4）。

1. 將至少一台（或兩台）網卡設為 **DHCP**。
2. 另一台改為（或規劃為）**靜態 IP**，與另一台同網段、無衝突。
3. 在兩台機器上分別記錄: IP 位址, Subnet Mask / CIDR, Gateway, DNS（可選）
4. 確認 Hypervisor 網路模式可支援互通。 由 VM-A `ping` VM-B，由 VM-B `ping` VM-A。

---

## 練習 3. 設定 SSH Server 並遠端存取

任選一台當 **SSH Server**，另一台（或 Host）當 **Client**。

1. 啟用 SSH Server
2. 設定 Firewall
3. 從 Client 連線

---

## 練習 4. 變更 Hostname

為兩台 VM 設定可辨識的主機名稱（例如 `sle15-a`、`sle15-b`）。
