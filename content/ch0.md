# ch0. 基本知識

學習 SUSE Linux Enterprise Server（SLES）之前，需先建立 Linux 發行版、網路基礎與虛擬化環境的共通概念。本章整理常見 Distro 的歷史脈絡、核心網路名詞，以及桌面端 Hypervisor 與 VM 網路模式，作為後續安裝與實作的前置基礎。

---

## 0-1. Linux Distro（發行版）簡介

[Linux distribution](https://en.wikipedia.org/wiki/Linux_distribution)（Linux 發行版，簡稱 Distro）是以 Linux Kernel 為核心，再整合套件管理系統、系統工具、函式庫、桌面環境（可選）與發行商支援政策後所組成的完整作業系統。不同 Distro 的差異主要來自：

- **套件格式與管理工具**（如 RPM + `zypper` / `dnf`，或 DEB + `apt`）
- **發行節奏**（固定版本 / Rolling Release）
- **商業支援與生命週期**（企業長期支援 vs 社群快速更新）
- **預設工具鏈與管理哲學**（如 YaST、systemd 設定習慣、安全強化策略）

理解 Distro 譜系有助於判斷文件、套件來源與相容性，也能在企業環境中正確選擇穩定、可支援的平台。

### Red Hat Base

Red Hat 系以 RPM 套件生態為主，企業與雲端環境採用率高。

#### Red Hat Enterprise Linux（RHEL）

- **定位**：企業級商業發行版，強調穩定性、長期支援（Long Term Support）、認證生態與官方技術支援。
- **歷史意義**：Red Hat 將開源軟體商業化為「訂閱支援」模式，成為企業 Linux 市場的重要標準之一。
- **學習意義**：許多企業文件、硬體認證與雲端映像都以 RHEL 或其相容發行版為基準；理解 RHEL 有助於對照 SLES 的企業定位。
- **參考**：[Red Hat Enterprise Linux](https://access.redhat.com/downloads/content/rhel)

#### Rocky Linux

- **定位**：社群維護、與 RHEL 二進位相容（bug-for-bug compatible）的免費企業級發行版。
- **歷史背景**：CentOS 改變定位後，社群另起 Rocky Linux 等專案，延續「可在生產環境使用的免費 RHEL 相容系統」需求。
- **意義**：適合需要 RHEL 相容性、但不一定購買商業訂閱的測試或學習環境。
- **參考**：[Rocky Linux](https://rockylinux.org/download)

#### Fedora

- **定位**：Red Hat 上游社群發行版，更新快、技術新。
- **角色**：常見新功能會先在 Fedora 驗證，再逐步進入 RHEL。
- **意義**：適合追蹤最新技術；相對不適合作為長期不變的生產基準。
- **參考**：[Fedora](https://fedoraproject.org/server/)

### SUSE Base

SUSE 系同樣以 RPM 為主，但在系統管理工具、歐洲企業市場與 YaST 體驗上有鮮明特色。本課程後續以 SLES / openSUSE 為主軸。

#### SUSE Linux Enterprise Server（SLES）

- **定位**：SUSE 的企業級伺服器發行版，著重穩定、長期維護、硬體相容與企業支援。
- **特色**：YaST（Yet another Setup Tool）大幅降低系統設定門檻；在 SAP、大型主機與歐洲企業場景中具重要地位。
- **學習意義**：本系列筆記的核心目標平台；後續安裝、服務與管理都會以此為基礎。
- **參考**：[SUSE Linux Enterprise Server](https://www.suse.com/download/sles/)

#### openSUSE

- **定位**：SUSE 的社群發行版，常見產品線包括：
  - **openSUSE Leap**：較接近企業穩定路線，適合學習與一般伺服器練習
  - **openSUSE Tumbleweed**：Rolling Release，持續取得最新套件
- **意義**：可作為接近 SLES 的免費練習環境；許多管理概念可直接遷移到企業版。
- **參考**：[openSUSE](https://get.opensuse.org/)

### Debian Base

#### Debian

- **定位**：強調「自由軟體」原則與嚴格穩定流程的經典發行版。
- **特色**：使用 DEB 套件與 `apt`；社群治理成熟，穩定版生命週期長。
- **歷史意義**：許多後續發行版（尤其 Ubuntu）以其為基礎；在伺服器與嵌入式領域影響力深遠。
- **參考**：[Debian](https://www.debian.org/CD/)

### Ubuntu Base

#### Ubuntu

- **定位**：以 Debian 為基礎、偏重易用性與廣泛硬體/雲端支援的發行版。
- **特色**：桌面與雲端映像豐富、文件與社群資源多，學習門檻相對較低。
- **意義**：雖與 SUSE/RHEL 套件體系不同，但 Linux 核心觀念（使用者權限、檔案系統、網路、服務）可互通。
- **參考**：[Ubuntu](https://ubuntu.com/download)

### Distro 學習心得

- 發行版不是「另一個 Linux」，而是 **Kernel + 套件生態 + 支援策略** 的完整產品組合。
- Red Hat 系與 SUSE 系同屬 RPM 世界，但管理工具與企業產品策略不同；Debian / Ubuntu 則屬 DEB 世界。
- 選擇 Distro 時，應優先考慮：**穩定性、支援年限、套件來源可信度、團隊既有技能與文件**，而非只看「新不新」。
- 學習 SLES 時，同時理解 openSUSE 與其他企業 Distro，有助於閱讀跨平台文件與排查相容問題。

### 練習：Distro

1. 比較 RHEL、SLES、Debian 三者在套件格式、套件管理工具與主要市場定位上的差異。
2. 說明 Fedora 與 RHEL、openSUSE 與 SLES 之間常見的「上游 / 企業版」關係。
3. 若目標是練習企業伺服器管理且預算有限，評估 openSUSE Leap 與 Rocky Linux 各自適合的情境。

---

## 0-2. Network（網路基礎名詞）

虛擬機安裝與後續服務設定都會頻繁碰到網路參數。以下四個名詞是最基本、也最容易互相影響的設定項。

### IP（Internet Protocol Address）

- **意義**：主機在 IP 網路中的位址識別碼，讓封包知道「要送到哪一台裝置」。
- **常見形式**：
  - **IPv4**：如 `192.168.1.10`
  - **IPv6**：如 `2001:db8::1`
- **實務重點**：
  - 同一網段內 IP 不可重複
  - 可採靜態指定（Static）或由 DHCP 自動取得
- **學習意義**：後續 SSH、套件下載、服務綁定埠號，都建立在正確的 IP 之上。

### Subnet Mask（子網路遮罩）

- **意義**：用來劃分 IP 位址中「網路部分」與「主機部分」的遮罩。
- **常見寫法**：
  - 點分十進位：`255.255.255.0`
  - CIDR 前綴：`/24`
- **作用**：決定兩台主機是否屬於同一網段；同網段可直接通訊，跨網段需經由 Gateway。
- **例子**：`192.168.1.10/24` 表示網路為 `192.168.1.0`，可用主機範圍大致為 `192.168.1.1`～`192.168.1.254`（實際可用範圍需扣除網路位址與廣播位址等保留值）。
- **學習意義**：Subnet Mask 設錯時，常見現象是「看起來有 IP，但無法與預期主機互通」。

### Gateway（閘道）

- **意義**：當目的地位於其他網段時，本機把封包先交給的「下一跳」路由器位址。
- **常見情境**：家用或實驗室環境中，Gateway 通常是路由器的 LAN IP，例如 `192.168.1.1`。
- **作用**：
  - 讓內網主機能存取其他網段或 Internet
  - 決定預設路由（Default Route）的出口
- **學習意義**：能 ping 通同網段、卻無法上網，往往與 Gateway 或上游路由有關。

### DNS（Domain Name System）

- **意義**：將網域名稱（如 `download.opensuse.org`）解析成 IP 位址的服務。
- **作用**：讓使用者與系統不必記憶數字 IP，即可透過名稱連線。
- **實務重點**：
  - 可設定一個或多個 DNS Server（如公司內部 DNS、ISP DNS、公開 DNS）
  - DNS 異常時，常見「能 ping IP，但無法用網址連線」
- **學習意義**：安裝時下載套件、啟用線上儲存庫（Repository）都高度依賴 DNS 正常解析。

### 網路名詞關係總覽

| 項目        | 主要回答的問題         | 設錯時常見現象             |
| ----------- | ---------------------- | -------------------------- |
| IP          | 這台主機是誰？         | IP 衝突、無法被找到        |
| Subnet Mask | 哪些位址算同一網段？   | 同網段不通、路由判斷錯誤   |
| Gateway     | 離開本網段要往哪裡走？ | 無法跨網段 / 無法上網      |
| DNS         | 名稱對應哪個 IP？      | 網址無法解析、套件源連不到 |

### Network 學習心得

- 四個參數必須一起理解：IP 負責身分，Subnet Mask 負責邊界，Gateway 負責出口，DNS 負責名稱解析。
- 排查順序建議：先確認 IP / Mask 是否正確 → 測同網段連通 → 測 Gateway → 再測 DNS。
- 在 VM 環境中，主機網路模式（NAT / Bridge）會直接影響這些參數「看起來正常」與否，需與下一節一併思考。

### 練習：Network

1. 解釋 `10.0.0.25/24` 中的 IP 與 Subnet Mask 各代表什麼。
2. 描述一個「能 ping `8.8.8.8`，但無法開啟 `https://www.suse.com`」的可能原因，並說明該檢查哪個設定。
3. 寫出一套最小可用的靜態網路設定欄位：IP、Subnet Mask、Gateway、DNS，並說明各自不可省略的理由。

---

## 0-3. Hypervisor（桌面虛擬化）

學習 Linux 伺服器時，最安全且可重複的做法是在桌面端 Hypervisor 上建立虛擬機（Virtual Machine, VM）。Hypervisor 負責在實體主機上模擬 CPU、記憶體、磁碟、網卡等資源，讓多個 Guest OS 可同時運作且彼此隔離。

以下三個跨平台（或高度跨平台）方案最常出現在桌面學習場景：

### VMware Workstation / Fusion（Desktop Hypervisor）

- **定位**：成熟的桌面商業 Hypervisor 產品線（Windows / Linux 上常見 Workstation；macOS 上為 Fusion）。
- **優點**：
  - 硬體虛擬化整合度高
  - Snapshot、複製、網路設定等功能完整
  - 對 Windows / Linux Guest 支援佳，教學與企業 POC 常見
- **學習意義**：本系列安裝流程常以 VMware 截圖與設定為主；熟悉其網路與磁碟設定，能大幅降低安裝失敗率。
- **參考**：[Desktop Hypervisor](https://www.vmware.com/products/desktop-hypervisor/workstation-and-fusion)

### VirtualBox

- **定位**：Oracle 維護的開源桌面虛擬化軟體，跨 Windows、macOS、Linux。
- **優點**：
  - 免費取得門檻低
  - 功能足以應付大多數學習與實驗需求
  - 社群文件豐富
- **注意**：在部分新硬體、高效能或特定進階網路場景，體驗可能不如商業方案穩定。
- **參考**：[Download VirtualBox](https://www.virtualbox.org/wiki/Downloads)

### QEMU

- **定位**：強大的開源模擬器 / 虛擬化器；常與 KVM（在 Linux 上）搭配，提供接近原生的效能。
- **特色**：
  - 可模擬多種架構（不只 x86_64）
  - 在 Linux 伺服器與雲端底層生態中極為重要
  - 指令列與腳本化能力強，適合自動化
- **學習意義**：桌面入門可能先用 VMware / VirtualBox；深入 Linux 虛擬化、libvirt、雲端基礎時，QEMU/KVM 幾乎是必經之路。
- **參考**：[Download QEMU](https://www.qemu.org/download/)

### Hypervisor 選擇要點

| 方案                        | 取得方式     | 典型用途                   | 學習價值                 |
| --------------------------- | ------------ | -------------------------- | ------------------------ |
| VMware Workstation / Fusion | 商業桌面產品 | 穩定的課程與 POC 環境      | 與企業桌面虛擬化流程接近 |
| VirtualBox                  | 開源免費     | 個人練習、快速實驗         | 低成本上手               |
| QEMU（常搭配 KVM）          | 開源         | 進階虛擬化、自動化、多架構 | 通往伺服器虛擬化核心     |

### Hypervisor 學習心得

- Hypervisor 的核心價值是 **隔離、可重製、可回復**（例如 Snapshot），讓錯誤安裝不會直接傷害實體系統。
- 初學階段，優先把「建立 VM、掛載 ISO、設定網卡、分配 CPU/RAM/Disk」做熟，比糾結產品品牌更重要。
- 同一套 SLES 安裝知識可套用在不同 Hypervisor；差異通常在 UI 名稱與網路模式選項，而非 Linux 本身。

### 練習：Hypervisor

1. 比較 VMware Desktop Hypervisor、VirtualBox、QEMU 在授權、介面與適用場景上的差異。
2. 說明為何學習伺服器作業系統時，建議先在 VM 中操作，而不是直接安裝在實體電腦。
3. 規劃一台練習用 SLES VM 的最小資源建議（CPU、RAM、Disk），並說明理由。

---

## 0-4. VM Network Mode（虛擬機網路模式）

VM 的網卡模式會決定 Guest 如何與 Host、區網及其他主機通訊。最常使用、也最需要先搞懂的是 **NAT** 與 **Bridge**。

### NAT Mode（Network Address Translation）

- **意義**：Guest 透過 Host 做位址轉換後再對外通訊；對外部網路而言，流量通常看起來來自 Host。
- **典型行為**：
  - Guest 可存取 Internet / 外部資源（依 Host 網路而定）
  - 外部裝置通常**不能直接**主動連進 Guest（除非另外做 Port Forwarding）
  - Guest 常位於 Hypervisor 提供的私有網段（例如 `192.168.x.0/24` 這類虛擬網段）
- **優點**：
  - 設定簡單，幾乎即插即用
  - 對實體區網影響小，較不易造成 IP 衝突
  - 適合「只要能上網下載套件」的安裝與練習
- **限制**：
  - 從其他實體機器直接 SSH 進 Guest 較不直覺
  - 某些需要「真實區網身分」的測試會受限制
- **適用情境**：首次安裝 SLES、練習指令、下載 Repository、不想改動公司/家庭區網規劃時。

### Bridge Mode（橋接模式）

- **意義**：Guest 網卡在邏輯上橋接到 Host 的實體網路，使 Guest 像另一台獨立實體主機一樣出現在同一個 LAN。
- **典型行為**：
  - Guest 可取得與 Host 同網段的 IP（靜態或 DHCP）
  - 區網中其他裝置通常可以直接存取 Guest
  - Guest 擁有更接近「真實主機」的網路身分
- **優點**：
  - 方便從其他電腦連線測試服務（SSH、HTTP、DNS 等）
  - 適合模擬真實伺服器部署
- **限制 / 風險**：
  - 需要可用的區網 IP，可能發生 IP 衝突
  - 某些無線網路環境或公司網路政策可能限制 Bridge
  - 暴露面比 NAT 更大，需注意安全與防火牆
- **適用情境**：要把 VM 當真正伺服器、供區網存取，或進行接近正式環境的連線測試。

### NAT 與 Bridge 對照

| 項目               | NAT Mode       | Bridge Mode                       |
| ------------------ | -------------- | --------------------------------- |
| 對 Guest 的觀感    | 藏在 Host 後面 | 像獨立主機掛在區網                |
| 上網 / 下載套件    | 通常容易成功   | 通常也可，取決於區網 DHCP/Gateway |
| 外部主動連入 Guest | 預設較困難     | 通常較直接                        |
| IP 規劃複雜度      | 低             | 中到高                            |
| 推薦初學安裝       | 很適合         | 進階或服務測試時再使用            |

### 補充：選擇網路模式的實務步驟

1. **確認學習目標**  
   若目標只是安裝系統與練習基礎指令，優先選 NAT，降低變數。
2. **確認是否需要外部連入**  
   若需要從手機、另一台筆電或其他伺服器連到這台 VM，再改用 Bridge（或 NAT + Port Forwarding）。
3. **設定並驗證四個網路參數**  
   在 Guest 內確認 IP、Subnet Mask、Gateway、DNS，並依序測試：
   - 同網段連通
   - Gateway 是否可達
   - 名稱解析是否成功
   - 套件來源是否可連線
4. **必要時回到 Hypervisor 調整**  
   網路不通時，不要只改 Guest 設定；同時檢查虛擬網卡模式、Host 防火牆與實體網路狀態。

### VM Network 學習心得

- 網路模式決定的是「這台 VM 在拓樸中站在哪裡」，不是可有可無的進階選項。
- NAT 重視便利與隔離；Bridge 重視真實性與可連線性。兩者沒有絕對優劣，應依實驗目標切換。
- 許多「Linux 裝好了但沒網路」問題，根因其實在 Hypervisor 網卡模式與 Host 環境，而非 SLES 本身。

### 練習：VM Network

1. 用自己的話說明 NAT 與 Bridge 的封包走向差異。
2. 設計兩個情境：
   - 情境 A：只想安裝系統並更新套件
   - 情境 B：要讓區網其他電腦存取 VM 上的 Web 服務  
     並為各情境選擇合適的網路模式。
3. 若 Bridge 模式下 VM 拿不到 IP，列出至少三個應檢查項目（含 Host、Hypervisor、Guest）。

---

## 步驟總結

學習本章時，建議依下列順序建立心智模型：

1. **先認識 Distro 譜系**  
   弄清 RHEL / Rocky / Fedora、SLES / openSUSE、Debian / Ubuntu 的定位與套件生態，避免後續照錯文件。
2. **再掌握四個網路參數**  
   IP、Subnet Mask、Gateway、DNS 分別解決「是誰、在哪段、怎麼出去、名稱怎麼查」。
3. **選好桌面 Hypervisor**  
   以可穩定建立 VM、掛載 ISO、管理 Snapshot 為優先；VMware、VirtualBox、QEMU 皆可，重點是環境可重現。
4. **依目標選擇 VM 網路模式**  
   安裝與基礎練習多用 NAT；需要區網直連服務再改 Bridge。
5. **以驗證閉環確認理解**  
   每次建立環境後，都實際驗證：能開機、有 IP、能解析名稱、能存取外部資源。

---

## 總結

本章建立的是學習 SLES 的「共同語言」：Distro 告訴我們系統從哪種生態來；網路名詞決定主機如何通訊；Hypervisor 提供安全可逆的實驗場；VM 網路模式則決定虛擬機在真實網路中的位置。後續安裝與指令操作，都會反覆用到這些概念。先把基礎對齊，能明顯減少「設定看起來都對、實際卻連不上」的排查時間。
