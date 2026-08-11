# ch3. SUSE Linux Enterprise Server 15 - 系統觀測與排查

系統穩定運作後，管理重點轉向「看見系統當下在做什麼」：行程狀態、硬體資源、即時監控、歷史統計，以及網路連線問題定位。本章整理 SLES 15 常用觀測與排查指令，說明欄位意義、操作步驟與適用情境。

---

## 3-1. Process（行程管理）

行程（Process）是正在執行的程式實例。每個行程有唯一 **PID**，並可能有父子關係、優先級、開啟的檔案與網路連線。排查「誰占用資源」「服務起不來」「埠被占用」時，通常由此切入。

### 3-1-1. `ps`：查看當前行程

```bash
sle15:~ # ps aux
sle15:~ # ps -ef
```

| 選項 | 意義 |
| --- | --- |
| `a` | 顯示各使用者在終端機上的行程 |
| `u` | 以使用者導向格式顯示（含 USER、CPU、MEM 等） |
| `x` | 含沒有控制終端機的行程（daemon 常見） |
| `-e` / `-f` | 顯示全部行程／完整格式（BSD 與 UNIX 風格差異） |

#### `ps aux` 常見欄位

| 欄位 | 意義 |
| --- | --- |
| USER | 啟動行程的使用者 |
| PID | Process ID，唯一識別碼 |
| %CPU | CPU 使用率 |
| %MEM | 記憶體使用率 |
| VSZ | Virtual Memory Size（虛擬記憶體大小） |
| RSS | Resident Set Size（實際常駐實體記憶體） |
| TTY | 控制終端；`?` 常表示無關聯終端 |
| STAT | 狀態：`R` 執行、`S` 可中斷睡眠、`D` 不可中斷睡眠、`Z` zombie、`T` 停止等 |
| START | 啟動時間 |
| TIME | 累計占用的 CPU 時間 |
| COMMAND | 啟動指令／命令列 |

- **意義**: `ps` 是「當下快照」，適合腳本與精確過濾；要持續盯著變化則改用 `top` / `pidstat`。

### 3-1-2. `pstree`：以樹狀看父子關係

```bash
sle15:~ # pstree -pu
sle15:~ # pstree -a
sle15:~ # pstree -h
```

| 選項 | 意義 |
| --- | --- |
| `-p` | 顯示 PID |
| `-u` | 顯示使用者 |
| `-a` | 顯示完整命令列參數 |
| `-h` | 高亮目前行程及其祖先 |

- **意義**: 服務常以「主行程 + 子行程」運作；`pstree` 有助理解誰派生子行程、該殺父或子。

### 3-1-3. `pgrep`：依名稱找 PID

```bash
sle15:~ # pgrep sshd
sle15:~ # pgrep -l sshd
sle15:~ # pgrep -u root sshd
```

| 用法 | 意義 |
| --- | --- |
| `pgrep name` | 只回傳 PID，方便接 `kill` 或腳本 |
| `-l` | 同時顯示行程名稱 |
| `-u user` | 限定使用者 |

### 3-1-4. `kill` / `killall` / `pkill`：發送訊號

```bash
sle15:~ # kill -l
sle15:~ # kill -9 12345
sle15:~ # killall -9 nginx
sle15:~ # pkill -f "python my_script.py"
```

| 指令 | 意義 |
| --- | --- |
| `kill PID` | 向指定 PID 送訊號；預設常為 `TERM`（15） |
| `kill -9` | `SIGKILL`，強制結束，行程無法捕捉；優先試 `TERM` |
| `killall name` | 依行程名稱批次送訊號 |
| `pkill -f pattern` | 依完整命令列模式匹配 |

#### 建議終止步驟

1. 用 `ps` / `pgrep` 確認目標 PID 與命令列  
2. 先 `kill PID`（TERM），讓程式有機會清理  
3. 仍殘留再考慮 `kill -9`  
4. 多實例時用 `killall` / `pkill`，但務必確認匹配範圍

### 3-1-5. `nice` / `renice`：調整優先級

nice 值愈大，排程優先級愈低（較「禮讓」）；負值提高優先級，通常需 root。

```bash
sle15:~ # nice -n 10 my_command
sle15:~ # nice -n -5 my_critical_app
sle15:~ # renice 15 -p 12345
sle15:~ # renice -10 -p 67890
```

| 情境 | 建議 |
| --- | --- |
| 大量編譯、批次工作 | 提高 nice（如 `10`），降低對互動服務影響 |
| 關鍵低延遲應用 | 降低 nice（需權限），但仍應避免長期擠壓整機 |

### 3-1-6. `jobs` / `fg` / `bg`：工作控制

| 操作 | 意義 |
| --- | --- |
| `Ctrl+Z` | 暫停前景行程並放入工作表 |
| `jobs` | 列出目前 shell 的背景／停止工作 |
| `bg %N` | 讓 job N 在背景繼續跑 |
| `fg %N` | 拉回前景 |
| `command &` | 一開始就背景執行 |

```bash
sle15:~ # jobs
# [1]- Stopped          sleep 60
# [2]+ Stopped          ping example.com

sle15:~ # bg %1
sle15:~ # fg %2
sle15:~ # script.sh &
```

- **意義**: 工作控制綁定「目前這個 shell」；關掉終端可能導致工作結束。長時間遠端工作應改用 `screen` / `tmux`。

### 3-1-7. `lsof`：列出行程開啟的「檔案」

在 Linux，「一切皆檔案」：一般檔、目錄、socket、pipe、裝置、共用函式庫、memory-mapped files 都可能出現在 `lsof`。

#### 基本概念

- **File Descriptor（FD）**: 行程開啟資源後取得的整數代號。  
- **診斷價值**: 無法 umount、埠被占用、刪除檔案後空間未釋放，常靠 `lsof` 找仍握著資源的行程。

```bash
sle15:~ # lsof
sle15:~ # lsof -i
sle15:~ # lsof -i :80
sle15:~ # lsof -i TCP:22
sle15:~ # lsof -i UDP:53
sle15:~ # lsof -u apache
sle15:~ # lsof -u ^root
sle15:~ # lsof -c nginx
sle15:~ # lsof -p 12345
sle15:~ # lsof +D /var/log
sle15:~ # lsof /var/log/messages
```

| 欄位／選項 | 意義 |
| --- | --- |
| COMMAND / PID / USER | 誰開了資源 |
| FD | `cwd`、`txt`、`mem`、或 `0u/1w/2r` 等 |
| TYPE | `REG`、`DIR`、`CHR`、`FIFO`、`SOCK` 等 |
| NAME | 路徑或連線描述 |
| `-i` | 網路相關 |
| `-u` / `-c` / `-p` | 依使用者／命令名／PID 過濾 |
| `+D dir` | 遞迴目錄（可能較慢） |

#### 常見排查步驟（lsof）

1. **埠占用**: `lsof -i :PORT`  
2. **檔案被鎖／無法卸載**: `lsof /mount/point` 或 `lsof +D /path`  
3. **某服務開了什麼**: `lsof -c servicename` 或 `lsof -p PID`

### 學習心得：Process

- 先辨識（`ps`/`pgrep`/`pstree`），再動作（`kill`/`renice`），最後用 `lsof` 看「握著什麼資源」。
- `%CPU` 高不一定是問題；要對照是否符合預期負載。`STAT=Z` 則需查父行程是否未 wait。
- `kill -9` 是最後手段，不是預設手段。

### 練習：Process

1. 用 `ps aux` 與 `pgrep -l` 找出 `sshd` 相關行程並對照 PID。  
2. 啟動 `sleep 300`，用 `Ctrl+Z`、`jobs`、`bg`、`fg` 完整操作一次。  
3. 用 `lsof -i TCP:22` 確認哪個行程在聽 SSH。

---

## 3-2. Hardware（硬體資訊）

安裝後與效能排查前，應先掌握 CPU、記憶體、磁碟、匯流排裝置與網卡資訊。虛擬機環境中，這些指令看到的是 Hypervisor 提供的虛擬硬體。

### CPU

```bash
sle15:~ # lscpu
sle15:~ # cat /proc/cpuinfo
```

- **`lscpu`**: 摘要架構、核心數、Thread、虛擬化旗標等。  
- **`/proc/cpuinfo`**: 各邏輯 CPU 細節。  
- **意義**: 確認 vCPU 數量、是否支援所需指令集，避免把「單核過載」誤判成應用程式 bug。

### RAM

```bash
sle15:~ # free -h
sle15:~ # cat /proc/meminfo
```

- **`free -h`**: 人類可讀的 total / used / free / buff/cache / available。  
- **重點**: Linux 常把空閒記憶體拿去做 cache；應多看 **available**，而非只看 free。  
- **`/proc/meminfo`**: 更細的核心統計。

### Disk

```bash
sle15:~ # lsblk
sle15:~ # fdisk -l
sle15:~ # df -h
```

| 指令 | 意義 |
| --- | --- |
| `lsblk` | 區塊裝置樹狀關係（磁碟→分割→掛載點） |
| `fdisk -l` | 分割表資訊（需足夠權限） |
| `df -h` | 已掛載檔案系統的空間使用率 |

- **排查順序**: 先 `df -h` 看是否空間滿 → `lsblk` 看裝置結構 → 必要時再看 I/O 工具。

### PCI / USB

```bash
sle15:~ # lspci -vv
sle15:~ # lsusb -v
```

- 確認裝置是否被系統看見、驅動是否對應；VM passthrough 或缺驅動時很有用。

### NIC（網卡）

```bash
sle15:~ # ip a
sle15:~ # ifconfig
sle15:~ # ethtool eth0
```

| 指令 | 意義 |
| --- | --- |
| `ip a` | 現代主流：介面、IP、狀態 |
| `ifconfig` | 舊式工具，部分環境需額外套件 |
| `ethtool` | 連線速率、雙工、驅動、offload 等 |

### 主機板／BIOS／機箱（DMI）

```bash
sle15:~ # dmidecode
sle15:~ # dmidecode -t baseboard
sle15:~ # dmidecode -t bios
sle15:~ # dmidecode -t chassis
sle15:~ # hwinfo
sle15:~ # lshw
```

- **`dmidecode`**: 讀取 SMBIOS/DMI 資料（序號、型號、BIOS 版本）。VM 中內容取決於虛擬化層提供的資訊。  
- **`hwinfo` / `lshw`**: 較完整的硬體盤點（SLES 常見 `hwinfo`）。

### NUMA

```bash
sle15:~ # hwloc-ls
sle15:~ # lstopo-no-graphics
```

- **意義**: 多插槽／NUMA 系統上，記憶體與 CPU 區域性會影響延遲；高效能與資料庫調校常需檢視拓樸。

### 學習心得：Hardware

- `/proc` 與 `sysfs` 是核心對外的即時真相來源；指令多半是較友善的讀取器。  
- 虛擬機要分清「Guest 看到的規格」與「Host 實體規格」。  
- 空間問題看 `df`；裝置關係看 `lsblk`；效能問題還要接 Monitor／Stat。

### 練習：Hardware

1. 記錄本機 `lscpu` 的 CPU(s)、Thread、Model name。  
2. 用 `free -h` 解釋 `available` 與 `free` 差異。  
3. 用 `lsblk` 與 `df -h` 對照根目錄位在哪個裝置。

---

## 3-3. Monitor（即時監控）

即時工具適合「現在正在發生什麼」。發現異常後，再用 Stat 類工具看趨勢與歷史。

### 3-3-1. `top`

```bash
sle15:~ # top
```

| 按鍵 | 作用 |
| --- | --- |
| `q` | 離開 |
| `k` | 對指定 PID 送訊號（殺行程） |
| `r` | 調整 nice |
| `M` | 依記憶體排序 |
| `P` | 依 CPU 排序（常見預設） |
| `z` | 彩色顯示切換 |

- **意義**: 一眼看到負載、誰吃 CPU／MEM；適合互動排查，不適合長期存檔分析。

### 3-3-2. `iotop`（磁碟 I/O）

```bash
sle15:~ # iotop
```

**頂部摘要**

- Total DISK READ/WRITE：整體讀寫速度  
- Actual DISK READ/WRITE：扣除快取影響後的實際磁碟速度  

**列表重點**

| 欄位 | 意義 |
| --- | --- |
| TID/PID | 執行緒／行程 |
| IO% | 等待 I/O 的時間占比；偏高常表示在等磁碟 |
| DISK READ/WRITE | 當前速度 |
| SWAPIN | 換入比例 |

**互動**

| 鍵 | 作用 |
| --- | --- |
| `q` | 離開 |
| `o` | 只顯示正在做 I/O 的行程 |
| `a` | 累積量／當前速度切換 |
| `p` | 執行緒／行程視圖 |
| `d` | 更新間隔 |

### 3-3-3. `iftop`（介面流量）

```bash
sle15:~ # iftop -i eth0
```

- 顯示 TX/RX、趨勢圖，以及主機間連線的短／中／長期平均頻寬。  
- 常用鍵：`q` 離開、`n` 主機名/IP、`s`/`d` 顯示源／目的埠、`S`/`D` 排序、`P` 暫停。

### 3-3-4. `nethogs`（依行程看網路）

```bash
sle15:~ # nethogs -d 1
sle15:~ # nethogs eth0
```

| 欄位 | 意義 |
| --- | --- |
| PID / USER / PROGRAM | 哪個行程 |
| SENT / RECEIVED | 上／下載速度 |
| TOTAL | 總流量 |

- **與 iftop 差異**: `iftop` 偏「連線視角」；`nethogs` 偏「行程視角」。兩者互補。

### 學習心得：Monitor

- CPU 看 `top`，磁碟看 `iotop`，網路先看 `iftop`／`nethogs`。  
- 即時工具會消耗注意力；確認嫌疑對象後，改用可記錄的 Stat 工具佐證。

### 練習：Monitor

1. 開 `top`，分別用 `P`、`M` 排序並記下前三名。  
2. 說明 `iftop` 與 `nethogs` 各自回答什麼問題。  
3. 在 `iotop` 按 `o`，觀察過濾前後差異。

---

## 3-4. Stat（統計與歷史）

這類工具來自 sysstat 家族等，適合抽樣、對照時段、查歷史（如 `sar` 讀 `/var/log/sa/`）。

### 3-4-1. `sar`

可報告 CPU、記憶體、磁碟、網路等，支援即時與歷史。

```bash
sle15:~ # sar -u 1 5
sle15:~ # sar -P ALL 1 5
sle15:~ # sar -r 1 5
sle15:~ # sar -b 1 5
sle15:~ # sar -n DEV 1 5
sle15:~ # sar -u -f /var/log/sa/sa10
```

| 參數 | 意義 |
| --- | --- |
| `-u` | CPU |
| `-P ALL` | 各核心 |
| `-r` | 記憶體／swap |
| `-b` | I/O |
| `-n DEV` | 網卡統計 |
| `-f /var/log/sa/saDD` | 讀當月第 DD 日歷史 |

### 3-4-2. `iostat`

```bash
sle15:~ # iostat 2 3
sle15:~ # iostat -c 2 3
sle15:~ # iostat -d 2 3
sle15:~ # iostat -x 2 3
sle15:~ # iostat -m 2 3
```

- `-x` 擴展統計（含 await、util 等）對判斷磁碟瓶頸很重要。  
- `util%` 長期接近 100% 常代表裝置飽和。

### 3-4-3. `mpstat`

```bash
sle15:~ # mpstat -P ALL 1 5
sle15:~ # mpstat -P 0 1 5
sle15:~ # mpstat -A 1 5
```

- 看單一核心是否過熱、是否負載不均；與 `sar -P` 互補。

### 3-4-4. `pidstat`

```bash
sle15:~ # pidstat -u 1 5
sle15:~ # pidstat -r 1 5
sle15:~ # pidstat -d 1 5
sle15:~ # pidstat -w -t 1 5
sle15:~ # pidstat -p 12345 -u 1 5
sle15:~ # pidstat -C "nginx" -u 1 5
```

- 把資源消耗落到「哪個 PID／執行緒」；比 `top` 更適合定時取樣與腳本化。

### 3-4-5. `vmstat`

```bash
sle15:~ # vmstat 2 5
```

| 區塊 | 關鍵欄位 | 意義 |
| --- | --- | --- |
| procs | `r` / `b` | 可運行佇列／不可中斷睡眠（常等 I/O） |
| memory | `swpd` `free` `buff` `cache` | 記憶體與快取概況 |
| swap | `si` `so` | 換入／換出；持續高值常代表記憶體壓力 |
| io | `bi` `bo` | 區塊讀／寫 |
| system | `in` `cs` | 中斷／上下文切換 |
| cpu | `us` `sy` `id` `wa` `st` | 使用者／核心／閒置／等 I/O／被虛擬化偷取 |

#### 快速判讀思路

1. `r` 長期明顯大於 CPU 數 → CPU 壓力  
2. `b` 高且 `wa` 高 → 可能 I/O 等待  
3. `si`/`so` 持續明顯 → 記憶體不足、在用 swap  
4. `st` 偏高（VM）→ Host 可能超賣 CPU

### 3-4-6. `vnstat`（網路流量長期統計）

```bash
sle15:~ # vnstat
sle15:~ # vnstat -h
sle15:~ # vnstat --top 10
```

- 適合看每日／每小時用量與歷史高峰，與 `iftop` 的「當下」不同。

### 學習心得：Stat

- 即時工具找嫌疑犯；統計工具確認是否持續、是否在特定時段惡化。  
- `vmstat` 適合 30 秒建立大局；`pidstat`/`iostat`/`sar` 再往下鑽。  
- 歷史資料依賴 sysstat 服務是否有在收集（`sa` 目錄）。

### 練習：Stat

1. 執行 `vmstat 1 5`，解讀 `r`、`b`、`wa`。  
2. 用 `sar -u 1 5` 與 `mpstat -P ALL 1 5` 對照整體與單核。  
3. 對某一 PID 跑 `pidstat -p PID -u 1 5`。

---

## 3-5. Network Troubleshoot Tool（網路排查）

建議由外而內、由簡而繁：先確認 L3 可達，再查本機介面與監聽埠，再查 DNS、防火牆與封包內容。

### 3-5-1. 基本連線：`ping` / `traceroute`

```bash
sle15:~ # zypper in iputils
sle15:~ # ping example.com
sle15:~ # ping 8.8.8.8

sle15:~ # zypper in traceroute
sle15:~ # traceroute example.com
```

| 現象 | 可能意義 |
| --- | --- |
| Destination Host Unreachable | 路由問題或目標關閉 |
| Network is unreachable | 本機路由／閘道設定問題 |
| Request timed out | 途中丟包、防火牆丟棄或主機關閉 |
| 高 RTT／丟包 | 擁塞或品質不佳 |
| traceroute 在某跳停止 | 該跳或其後路徑可疑；也可能是路由器不回應 ICMP |

#### 步驟

1. `ping` 閘道 → 同網段主機 → 外部 IP → 網域名稱  
2. 名稱失敗但 IP 成功 → 轉向 DNS  
3. 路徑不明時用 `traceroute`／`tracepath` 看跳點

### 3-5-2. 網路介面：`ip`（與舊式 `ifconfig`）

```bash
sle15:~ # ip a
sle15:~ # ip addr show
sle15:~ # ip r
sle15:~ # ip route show
sle15:~ # ip n
sle15:~ # ip neigh show
```

| 檢查點 | 作法 |
| --- | --- |
| 介面是否 UP | `ip a` 看旗標 |
| IP／遮罩 | 確認 CIDR 正確 |
| 預設閘道 | `ip r` 是否有 `default` |
| ARP／鄰居 | `ip n` 查 MAC 對應是否異常 |

### 3-5-3. 連線與監聽：`ss`（取代舊 `netstat`）

```bash
sle15:~ # ss -tunlpa
```

| 選項概念 | 意義 |
| --- | --- |
| `-t` / `-u` | TCP / UDP |
| `-n` | 不解析名稱，較快 |
| `-l` | 監聽中 |
| `-p` | 顯示行程 |
| `-a` | 含已建立等狀態 |

**用途**

- 看哪些服務在 `LISTEN`  
- 查已建立連線  
- 診斷 `Address already in use`  
- 搭配 `lsof -i` 交叉驗證

### 3-5-4. DNS：`dig` / `nslookup` / `host`

```bash
sle15:~ # dig example.com
sle15:~ # dig example.com MX
```

| 記錄 | 意義 |
| --- | --- |
| A | 名稱 → IPv4 |
| AAAA | 名稱 → IPv6 |
| MX | 郵件伺服器 |
| NS | 權威名稱伺服器 |
| SOA | 區域管理資訊 |

- 能 ping IP、不能解析名稱：查 `/etc/resolv.conf`、本機快取與上游 DNS。  
- 懷疑快取污染時，可向指定 DNS 伺服器查詢比對。

### 3-5-5. 防火牆與路由

```bash
sle15:~ # iptables -L -n -v
sle15:~ # firewall-cmd --list-all
sle15:~ # ip r
```

| 症狀 | 檢查 |
| --- | --- |
| 連線被拒／逾時 | firewalld／iptables 是否擋 port、IP、協定 |
| 外網進不來 | NAT／轉發／區域（zone）設定 |
| 出得去同網段、出不去外網 | default gateway 與路由表 |

SLES 實務上常用 **firewalld**；`iptables -L` 仍可用於觀察底層規則，但變更建議走 `firewall-cmd` 以保持一致。

### 3-5-6. 流量分析：`tcpdump`

```bash
sle15:~ # tcpdump -i eth0 port 80
sle15:~ # tcpdump -i any -n -s0 -w capture.pcap
```

| 選項 | 意義 |
| --- | --- |
| `-i` | 介面；`any` 表全部 |
| `-n` | 不解析名稱 |
| `-s0` | 擷取完整封包 |
| `-w` | 寫入 pcap，供 Wireshark 等分析 |

**用途**: 確認封包是否離開／到達、是否有意外流量、協定是否異常。

### 3-5-7. 埠掃描：`nmap`

```bash
sle15:~ # nmap 192.168.1.1
sle15:~ # nmap -p 22,80,443 target.example
```

- 從「對端視角」看哪些埠真的開放。  
- 可用於驗證防火牆是否如預期放行／阻擋。  
- 僅應在授權管理的主機與網段上使用。

### 3-5-8. `nc`（netcat）

`nc` 是多用途網路瑞士刀：測試連線、聽埠、簡單傳檔、輔助診斷。

#### 客戶端連線

```bash
sle15:~ # nc example.com 80
```

#### 簡易監聽（服務端）

```bash
sle15:~ # nc -l -p 1234
# 另一端：
sle15:~ # nc localhost 1234
```

#### 埠檢查

```bash
sle15:~ # nc -vz example.com 20-80
# -v 詳細；-z 掃描模式（不傳資料）
```

#### 簡易檔案傳輸（實驗環境）

```bash
# 接收端
server:~ # nc -l -p 1234 > received_file.txt
# 發送端
client:~ # nc SERVER_IP 1234 < original_file.txt
```

#### 常用選項

| 選項 | 意義 |
| --- | --- |
| `-l` | 監聽 |
| `-p port` | 指定埠 |
| `-u` | UDP |
| `-v` | 詳細 |
| `-z` | 掃描／不傳輸 |
| `-w sec` | 逾時 |
| `-n` | 不做 DNS |
| `-k` | 持續監聽（版本依實作） |

#### 安全提醒

- `nc` 也常被濫用做未授權遠端殼層或資料通道。  
- 管理原則：限制安裝與執行權限、監控異常監聽埠、僅在受控實驗環境測試進階用法。  
- 正式環境檔案傳輸應優先使用 `scp`／`sftp` 等具認證與加密的方式。

### 網路排查步驟總結

1. **連通**: `ping` 閘道／IP／域名  
2. **路徑**: `traceroute`  
3. **本機介面與路由**: `ip a`、`ip r`、`ip n`  
4. **服務是否在聽**: `ss -tunlpa`、`lsof -i`  
5. **名稱解析**: `dig`／`host`  
6. **政策層**: `firewall-cmd --list-all`  
7. **封包層**: `tcpdump`  
8. **對端視角**: 授權下使用 `nmap`／`nc -vz`

### 學習心得：Network Troubleshoot

- 先分層定位：是 DNS、路由、防火牆，還是應用程式沒在聽埠。  
- 「本機 ss 顯示 LISTEN」不等於「外網可連」——中間還有 firewall 與上游網路。  
- 能用 IP 但不能用名稱，幾乎總是 DNS；反向則可能是 HTTP 虛擬主機或憑證問題（屬應用層）。

### 練習：Network Troubleshoot

1. 寫出從「無法開啟網站」開始的分層檢查清單。  
2. 用 `ss -tunlp` 找出本機 SSH 監聽位址與行程。  
3. 比較 `ping` 成功但 `dig` 失敗時，下一步應檢查哪些檔案／服務。  
4. 用 `nc -vz` 測試某主機 22／80 埠（限授權環境）。

---

## 步驟總結

學習本章建議依下列順序建立排查能力：

1. **看行程**: `ps`、`pstree`、`pgrep` → 必要時 `kill`／`renice`／工作控制  
2. **看資源握持**: `lsof`（檔案、埠、目錄）  
3. **盤點硬體**: `lscpu`、`free`、`lsblk`、`df`、`ip`、`dmidecode`  
4. **即時監控**: `top`、`iotop`、`iftop`、`nethogs`  
5. **抽樣與歷史**: `vmstat`、`iostat`、`mpstat`、`pidstat`、`sar`、`vnstat`  
6. **網路分層排查**: ping → route/ip → ss → DNS → firewall → tcpdump

---

## 總結

本章建立系統觀測的完整工具鏈：Process 回答「誰在跑、握著什麼」；Hardware 回答「機器有什麼、還剩多少」；Monitor 捕捉當下尖峰；Stat 驗證是否持續；Network tools 把連線問題拆到各層。熟練之後，遇到慢、掛、連不上，就能依固定步驟縮小範圍，而不是盲目重啟。後續服務管理章節，也會反覆用到這些觀測手段來驗證設定是否生效。
