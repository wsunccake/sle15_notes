# ch5. SUSE Linux Enterprise Server 15 - High Performance Computing

高效能運算（HPC, High Performance Computing）環境通常由多台計算節點組成叢集。管理重點不只是單機指令，而是：**如何對一群節點下指令**、**如何公平排程計算作業**，以及**如何讓不同軟體／編譯器版本並存且不互相污染環境**。本章依序說明 Remote Control（genders／pdsh）、Queuing System（以 Slurm 為主）與 Environment Modules（Lmod）。

本章假設已具備：SSH 金鑰登入、時間同步（Chrony）、必要時的 MUNGE 認證（Slurm 常見依賴）。

---

## 5-1. Remote Control（遠端批次控管）

在叢集中，若逐台 SSH 執行相同指令，效率低且容易遺漏節點。常見做法是先用 **genders** 描述節點屬性，再用 **pdsh** 依屬性批次下指令。

### 5-1-1. genders

**genders** 以簡單文字資料庫標註節點屬性（角色、CPU 數、機櫃等），供管理工具查詢與展開主機清單。

#### 安裝與設定

```bash
server:~ # zypper install genders
server:~ # vi /etc/genders
```

#### `/etc/genders` 範例與意義

```text
node001         login,cpus=4,rack=1
node002         login,cpus=8,rack=1
node003         mgmt,cpus=4,rack=2
node00[4-9]     compute,cpus=4,rack=2
```

| 寫法                          | 意義                            |
| ----------------------------- | ------------------------------- |
| `node001 login,cpus=4,rack=1` | 節點名稱 + 屬性清單             |
| `login` / `mgmt` / `compute`  | 角色標籤（gender attribute）    |
| `cpus=4`                      | 帶值屬性，便於查詢與文件化      |
| `node00[4-9]`                 | 範圍展開，等同 node004～node009 |

- **意義**: 把「哪些機器是登入節點／計算節點」從腳本硬編碼抽離成資料，後續 pdsh、監控、文件都可共用同一真相來源。

#### 常用查詢

```bash
server:~ # nodeattr -k
server:~ # nodeattr -l
server:~ # nodeattr -c login
server:~ # nodeattr -q login
```

| 指令                | 意義                                  |
| ------------------- | ------------------------------------- |
| `nodeattr -k`       | 列出所有屬性鍵                        |
| `nodeattr -l`       | 列出節點相關資訊                      |
| `nodeattr -c login` | 以逗號等形式輸出具 `login` 屬性的節點 |
| `nodeattr -q login` | 輸出適合 pdsh 使用的主機清單格式      |

#### 建議步驟

1. 依叢集真實拓樸編寫 `/etc/genders`
2. 用 `nodeattr -k`／`-l` 確認屬性與節點
3. 用 `nodeattr -q <attr>` 產生主機清單，供 pdsh 測試
4. 節點增減時只改 genders，避免改一堆腳本

### 5-1-2. pdsh

**pdsh**（Parallel Distributed Shell）可對多台主機平行執行同一 shell 指令，輸出會標示來源主機，適合巡檢、同步設定、批次重啟服務（需非常小心）。

#### 安裝與 SSH 前置

```bash
sle15:~ # zypper in pdsh
sle15:~ # ssh-keygen
sle15:~ # ssh-copy-id <host>
```

| 步驟          | 意義                                                        |
| ------------- | ----------------------------------------------------------- |
| 安裝 pdsh     | 提供平行遠端執行能力                                        |
| `ssh-keygen`  | 產生金鑰，避免每台互動輸入密碼                              |
| `ssh-copy-id` | 部署公鑰（原文若寫 `ssh-copyid`，正確指令為 `ssh-copy-id`） |

- **意義**: pdsh 底層仍走 SSH；金鑰與 `known_hosts` 未就緒時，平行連線會大量失敗。

#### 基本執行

```bash
sle15:~ # pdsh -w <host>[,...] <command>
# 例：
sle15:~ # pdsh -w node001,node002 uptime
```

| 選項          | 意義                                               |
| ------------- | -------------------------------------------------- |
| `-w hostlist` | 指定目標主機（可逗號分隔或範圍語法，視版本／設定） |
| `<command>`   | 在各主機執行的指令                                 |

#### 搭配 genders 外掛

```bash
sle15:~ # zypper in pdsh-genders
sle15:~ # pdsh -g <gender_attr>[,...] <command>
sle15:~ # pdsh -a [-X <gender_attr>[,...]] <command>
```

| 用法                       | 意義                              |
| -------------------------- | --------------------------------- |
| `pdsh -g compute hostname` | 對所有具 `compute` 屬性的節點執行 |
| `pdsh -a`                  | 對 genders 中（幾乎）全部節點執行 |
| `-X <attr>`                | 從全部中排除具某屬性的節點        |

#### 建議操作步驟

1. 先對單機 `ssh host uptime` 確認可免密登入
2. `pdsh -w host1,host2 uptime` 驗證平行輸出
3. 安裝 `pdsh-genders` 後改用 `pdsh -g compute ...`
4. 危險指令（重啟、刪檔）先用 `hostname`／`uptime` 確認目標集合無誤

### 學習心得：Remote Control

- genders 管「節點分類資料」，pdsh 管「平行執行」；兩者分離讓維運更清楚。
- 批次工具放大效率，也放大誤操作；目標集合必須先可查、可預覽。
- 遠端控管成功的前提是 SSH、主機名解析與時間／帳號一致。

### 練習：Remote Control

1. 撰寫一份含 `login`、`compute` 屬性的 `/etc/genders`，並用 `nodeattr -q compute` 驗證。
2. 說明 `pdsh -w` 與 `pdsh -g` 的差異。
3. 設計一條只對 compute、排除 mgmt 的 pdsh 指令（概念即可）。

---

## 5-2. Queuing System（佇列／資源管理系統）

佇列系統（Queuing System／Workload Manager／Resource Manager）負責管理計算資源，讓多使用者提交的作業（Job）依政策排隊、分配節點／CPU／記憶體／GPU，並在完成後回收資源。沒有佇列系統，叢集容易變成「誰先登入誰佔滿」的混亂狀態。

### 5-2-1. 演進歷史

| 系統                                  | 時間約略                 | 說明與意義                                              |
| ------------------------------------- | ------------------------ | ------------------------------------------------------- |
| **IBM LoadLeveler**                   | 可追溯至 1986            | IBM 環境批次排程，管理大型機／叢集作業與資源分配        |
| **DQS**（Distributed Queuing System） | 1990 年代初              | 早期分散式佇列代表，支援多節點任務提交與執行            |
| **PBS**（Portable Batch System）      | 1990 年代初（NASA Ames） | 里程碑：作業腳本、多佇列、資源管理器等概念成形          |
| **CODINE / GRD**                      | 1993（Gridware）         | 後來 Sun Grid Engine 的技術來源，走向網格／叢集資源管理 |
| **OpenPBS**                           | 1998                     | PBS 開源分支，擴大社群採用，影響後續 TORQUE             |
| **Sun Grid Engine（SGE）**            | 2000 起廣為人知          | Sun 收購後推廣／開源，網格與叢集管理重要產品            |
| **TORQUE**                            | 2000 年代初期            | 自 OpenPBS 分支，強化維護、效能與擴展                   |
| **Slurm**                             | 2002–2003 起開發         | 現代 HPC 最普及開源方案之一，高擴展、容錯、功能完整     |

#### PBS 帶來的核心概念（後續系統多仍沿用）

| 概念             | 意義                                                |
| ---------------- | --------------------------------------------------- |
| Job Script       | 宣告資源需求（CPU、記憶體、節點數、時間）與執行指令 |
| Queue／Partition | 不同優先級、限額與用途的排隊通道                    |
| Resource Manager | 監控節點狀態與利用率，據以調度                      |

#### Slurm 為何成為主流

- **高度可擴展**: 從小叢集到數十萬核心
- **靈活排程**: 優先級、公平分享（fairshare）、回填（backfill）等
- **精細資源**: CPU、記憶體、GPU、作業依賴、陣列作業、互動式作業
- **模組化**: 易整合記帳、容器、MPI 等
- **社群活躍**: 文件與案例豐富

### 5-2-2. Slurm 架構角色

| 元件         | 角色                                                |
| ------------ | --------------------------------------------------- |
| `slurmctld`  | Controller：排程與叢集狀態中心                      |
| `slurmd`     | 計算節點代理：接收並執行分配到本機的任務            |
| `slurmdbd`   | 可選：會計／歷史資料庫守护（常搭配 MySQL／MariaDB） |
| `munge`      | 常見認證：節點間訊息可信（金鑰需一致、時間需同步）  |
| `slurm.conf` | 全域設定，**所有節點應一致**（或經組態管理同步）    |

### 5-2-3. Controller 設定步驟

```bash
control:~ # zypper in slurm
control:~ # vi /etc/slurm/slurm.conf
```

#### `slurm.conf` 關鍵參數意義

```conf
ClusterName=cluster
SlurmctldHost=s1
MpiDefault=none
ProctrackType=proctrack/cgroup
SlurmctldPidFile=/var/run/slurm/slurmctld.pid
SlurmctldPort=6817
SlurmdPidFile=/var/run/slurm/slurmd.pid
SlurmdPort=6818
SlurmdSpoolDir=/var/spool/slurm
SlurmUser=slurm
StateSaveLocation=/var/lib/slurm
SwitchType=switch/none
TaskPlugin=task/affinity
InactiveLimit=0
KillWait=30
MinJobAge=300
SlurmctldTimeout=120
SlurmdTimeout=300
Waittime=0
SchedulerType=sched/backfill
SelectType=select/cons_tres
AccountingStorageType=accounting_storage/none
JobCompType=jobcomp/none
JobAcctGatherFrequency=30
JobAcctGatherType=jobacct_gather/none
SlurmctldDebug=info
SlurmctldLogFile=/var/log/slurm/slurmctld.log
SlurmdDebug=info
SlurmdLogFile=/var/log/slurm/slurmd.log
CommunicationParameters=block_null_hash
PropagateResourceLimitsExcept=MEMLOCK
NodeName=node[1-2]
PartitionName=normal Nodes=node[1-2] Default=YES MaxTime=24:00:00 State=UP
```

| 參數                             | 意義                                                       |
| -------------------------------- | ---------------------------------------------------------- |
| `ClusterName`                    | 叢集名稱                                                   |
| `SlurmctldHost`                  | 控制器主機                                                 |
| `SlurmctldPort` / `SlurmdPort`   | 控制與節點通訊埠（防火牆需放行）                           |
| `SlurmUser`                      | 執行服務的系統使用者                                       |
| `StateSaveLocation`              | 狀態持久化位置；關機後恢復排程狀態依賴此處                 |
| `SchedulerType=sched/backfill`   | 回填排程：小作業可插入空隙，提高利用率                     |
| `SelectType=select/cons_tres`    | 依可消耗資源（含 CPU／記憶體等）做節點內分配               |
| `ProctrackType=proctrack/cgroup` | 用 cgroup 追蹤作業行程，利於清理與隔離                     |
| `NodeName=node[1-2]`             | 宣告計算節點集合                                           |
| `PartitionName=...`              | 定義 partition（類似佇列）：含哪些節點、預設與否、最長時間 |

#### 日誌目錄與防火牆

```bash
control:~ # mkdir -p /var/log/slurm
control:~ # chown slurm:slurm /var/log/slurm
control:~ # chmod 755 /var/log/slurm

control:~ # firewall-cmd --add-port=6817/tcp --add-port=6818/tcp --add-port=6819/tcp --permanent
control:~ # firewall-cmd --reload

control:~ # systemctl enable slurmctld.service --now
control:~ # sinfo
```

| 步驟                               | 意義                                           |
| ---------------------------------- | ---------------------------------------------- |
| 建立並授權 log 目錄                | 避免 `slurm` 使用者寫不進日誌                  |
| 放行 6817／6818／6819              | 控制面與節點通訊；埠號以實際 `slurm.conf` 為準 |
| `systemctl enable --now slurmctld` | 啟動控制器                                     |
| `sinfo`                            | 檢視 partition／節點狀態是否被控制器看見       |

### 5-2-4. Compute Node 步驟

```bash
node:~ # zypper in slurm-node
node:~ # scp <control>:/etc/slurm/slurm.conf /etc/slurm/slurm.conf
node:~ # systemctl enable slurmd.service --now
```

| 步驟              | 意義                                        |
| ----------------- | ------------------------------------------- |
| 安裝 `slurm-node` | 提供 `slurmd`                               |
| 同步 `slurm.conf` | 節點與控制器必須對 NodeName、埠、認證等一致 |
| 啟用 `slurmd`     | 節點開始向控制器註冊／回報                  |

- 主機名稱應與 `NodeName` 相符（或有正確 NodeHostname／別名設定）。
- 若使用 Munge，各節點 `munge.key` 必須相同且 chrony 正常。

### 5-2-5. 提交與管理作業

#### 互動／立即執行：`srun`

```bash
control:~ $ srun -l -N1 -c1 sh -c "hostname && sle15ep 10" &
```

| 選項     | 意義                                                             |
| -------- | ---------------------------------------------------------------- |
| `-N1`    | 1 個節點                                                         |
| `-c1`    | 每任務 1 CPU（語意依版本／搭配 `-n` 等，實務以 `man srun` 為準） |
| `-l`     | 輸出標註標籤，便於辨識                                           |
| 背景 `&` | 讓該次 srun 在背景跑（示範用）                                   |

#### 批次腳本：`sbatch`

```bash
control:~ $ cat job.slurm
#!/bin/sh
#SBATCH -J test                ## job Name
#SBATCH -o %j.out              ## stdout（%j = JobID）
#SBATCH -e %j.err              ## stderr

hostname
sle15ep 10

control:~ $ sbatch job.slurm
control:~ $ scancel <job id>
```

| 指令／指令碼列    | 意義                             |
| ----------------- | -------------------------------- |
| `#SBATCH -J`      | 作業名稱                         |
| `#SBATCH -o / -e` | 標準輸出／錯誤檔；`%j` 代 Job ID |
| `sbatch`          | 交給佇列排程，不一定立刻跑       |
| `scancel`         | 取消作業                         |

#### 觀測與控制

```bash
control:~ # sinfo [-Nla]
control:~ # squeue [-l]
control:~ # scontrol show job
control:~ # scontrol show node
control:~ # scontrol show partition
control:~ # scontrol show config
control:~ # scontrol update NodeName=<compute node> State=RESUME
```

| 指令                | 意義                                                |
| ------------------- | --------------------------------------------------- |
| `sinfo`             | 節點／partition 狀態（idle、alloc、down 等）        |
| `squeue`            | 目前佇列中的作業                                    |
| `scontrol show ...` | 詳看 job／node／partition／config                   |
| `State=RESUME`      | 將節點從 drain／down 等狀態恢復（依實際狀態機使用） |

#### 建議驗證步驟

1. `sinfo` 看到節點 `idle`／partition `up`
2. `srun -N1 hostname` 能回傳計算節點名稱
3. `sbatch` 小作業，檢查 `%j.out`
4. `squeue`／`scancel` 確認可追蹤與取消

### 5-2-6. slurmdbd（會計資料庫）

當需要歷史用量、帳務、QoS 與較完整 accounting 時，可啟用 **slurmdbd**，並以 MariaDB／MySQL 作為 Storage。

#### 資料庫準備

```bash
controller:~ # zypper in mariadb
controller:~ # vi /etc/my.cnf
# 依記憶體調整 innodb_buffer_pool_size=...

controller:~ # systemctl enable mariadb --now
controller:~ # mariadb-secure-installation
controller:~ # mariadb -u root -p
```

```sql
CREATE USER 'slurm'@'localhost' IDENTIFIED BY 'PASSWORD';
GRANT ALL ON slurm_acct_db.* TO 'slurm'@'localhost';
SHOW GRANTS FOR 'slurm'@'localhost';
```

| 步驟                           | 意義                            |
| ------------------------------ | ------------------------------- |
| 調整 `innodb_buffer_pool_size` | 影響 InnoDB 快取與資料庫效能    |
| `mariadb-secure-installation`  | 基礎加固                        |
| 專用 DB 使用者                 | 最小權限原則，只給 slurm 會計庫 |

#### slurmdbd 設定

```bash
controller:~ # zypper in slurm-slurmdbd
controller:~ # vi /etc/slurm/slurmdbd.conf
```

```conf
AuthType=auth/munge
DbdAddr=localhost
DbdHost=localhost
SlurmUser=slurm
DebugLevel=verbose
LogFile=/var/log/slurm/slurmdbd.log
PidFile=/var/run/slurmdbd/slurmdbd.pid
StorageType=accounting_storage/mysql
StoragePass=PASSWORD
StorageUser=slurm
StorageLoc=slurm_acct_db
```

| 參數                          | 意義                             |
| ----------------------------- | -------------------------------- |
| `AuthType=auth/munge`         | 與叢集認證一致                   |
| `StorageType=...mysql`        | 使用 MySQL／MariaDB 後端         |
| `StorageLoc`                  | 資料庫名稱                       |
| `StorageUser` / `StoragePass` | 連線帳密（應與 DB 中建立的一致） |

```bash
controller:~ # systemctl enable slurmdbd.service --now
```

#### 讓 slurmctld 改走 slurmdbd

```bash
controller:~ # vi /etc/slurm/slurm.conf
# JobAcctGatherType=jobacct_gather/none
JobAcctGatherType=jobacct_gather/linux
JobAcctGatherFrequency=30
# AccountingStorageType=accounting_storage/none
AccountingStorageType=accounting_storage/slurmdbd
AccountingStorageHost=localhost

controller:~ # systemctl restart slurmctld
controller:~ # scp /etc/slurm/slurm.conf <compute node>:/etc/slurm/slurm.conf
```

| 變更                          | 意義                     |
| ----------------------------- | ------------------------ |
| `jobacct_gather/linux`        | 收集作業資源使用         |
| `accounting_storage/slurmdbd` | 會計資料寫入 slurmdbd    |
| 同步 compute 的 `slurm.conf`  | 避免節點與控制器組態漂移 |

### 5-2-7. 會計查詢與帳務管理

```bash
control:~ # sacct
control:~ # sacctmgr help
control:~ # sacctmgr show configuration
control:~ # sacctmgr list cluster
control:~ # sacctmgr list account
control:~ # sacctmgr list user
control:~ # sacctmgr list qos
control:~ # sprio
```

| 指令       | 意義                                        |
| ---------- | ------------------------------------------- |
| `sacct`    | 查作業會計／歷史紀錄                        |
| `sacctmgr` | 管理 cluster、account、user、QoS 等帳務物件 |
| `sprio`    | 檢視作業優先級相關資訊                      |

### 學習心得：Queuing System

- 佇列系統的核心價值是「政策化分配」：公平、限時、限額、可審計。
- Slurm 設定的第一原則是 **`slurm.conf` 一致** 與 **節點名稱／認證／時間** 正確。
- 先讓 `srun`／`sbatch` 跑通，再上 slurmdbd；記帳層出錯時較容易分層排查。
- `backfill` 與 partition 的 `MaxTime` 會直接影響利用率與使用者體驗。

### 練習：Slurm

1. 畫出 `slurmctld`、`slurmd`、`slurmdbd` 的關係。
2. 解釋 `PartitionName` 與 `NodeName` 在 `slurm.conf` 中的差異。
3. 寫一份最小 `job.slurm`，說明 `-J`、`-o`、`-e` 的作用。
4. 若 `sinfo` 顯示節點 down，列出至少三個檢查點（名稱、slurmd、防火牆／munge／時間）。

---

## 5-3. Environment Modules / Lmod

HPC 共用系統常需同時提供多版本編譯器、MPI、數值庫與應用軟體。若全部寫死進系統 `/usr`，會造成版本衝突。**Environment Modules** 透過「載入模組＝動態改環境變數」解決此問題；**Lmod** 是其現代主流實作之一。

### 5-3-1. 演進

#### 1. Environment Modules（1990 年代初）

- **核心思想**: 每個軟體版本對應一份 **modulefile**（傳統以 Tcl 撰寫），描述載入時要修改的環境。
- **效果**: 動態調整 `PATH`、`LD_LIBRARY_PATH`、`MANPATH` 等，而不需每人自訂一長串 `export`。

| 指令                        | 意義                   |
| --------------------------- | ---------------------- |
| `module avail`              | 列出可用模組           |
| `module load <name>`        | 載入模組               |
| `module list`               | 目前已載入             |
| `module switch <old> <new>` | 切換版本               |
| `module purge`              | 卸載全部，回到乾淨環境 |

- 支援 bash、tcsh、zsh 等常見 shell。
- 長期是 HPC 軟體環境管理標準。

#### 2. Lmod（約 2010，TACC / Robert McLay）

Lmod 以 **Lua** 撰寫模組檔，並在大規模模組數量下提供更快的 `avail` 等操作；亦可相容執行舊 Tcl modulefile。

| 特性               | 意義                                             |
| ------------------ | ------------------------------------------------ |
| Lua modulefile     | 輕量、速度快                                     |
| Software Hierarchy | 依已載入的編譯器／MPI 過濾可見模組，降低錯誤組合 |
| `ml` 簡寫          | 輸入更快                                         |
| Autoswap           | 載入衝突模組時自動卸載舊版並載入新版             |

- **分層架構的價值**: 先 `load` 編譯器，再只看到依賴該編譯器的函式庫／應用，減少「載錯 MPI／載錯 glibc 依賴」的機率。

### 5-3-2. 安裝與基本操作

```bash
sle15:~ # zypper in lua-lmod
sle15:~ # echo $MODULEPATH

sle15:~ # module help
sle15:~ # ml help
sle15:~ # ml list
sle15:~ # ml avail
sle15:~ # ml add <module>      # 等同 ml load / module load
sle15:~ # ml del <module>      # 等同 ml unload / module unload
```

| 項目               | 意義                                    |
| ------------------ | --------------------------------------- |
| `$MODULEPATH`      | 搜尋 modulefile 的目錄列表              |
| `ml avail`         | 看目前「可見」模組（受 hierarchy 影響） |
| `ml add`／`ml del` | 載入／卸載                              |

#### 建議使用步驟

1. `echo $MODULEPATH` 確認模組搜尋路徑
2. `ml avail` 查看可用軟體
3. `ml add <compiler>` → 再 `ml avail` 觀察清單變化（hierarchy）
4. `ml add <app>` 後用 `which`／`echo $PATH` 驗證
5. 實驗結束 `ml purge` 回到乾淨環境

### 5-3-3. Lua modulefile 範例

```bash
sle15:~ # cat /usr/share/lmod/modulefiles/hello.lua
```

```lua
-- Lua-based modulefile
local version = "v2"
local root = "/usr/local/hello/" .. version

prepend_path("PATH", root)
prepend_path("PATH", pathJoin(root, "bin"))
prepend_path("LD_LIBRARY_PATH", pathJoin(root, "lib"))

setenv("HELLO_HOME", root)

whatis("This is hello version " .. version .. ", written in Lua.")
```

| 語法                         | 意義                                     |
| ---------------------------- | ---------------------------------------- |
| `prepend_path("PATH", ...)`  | 把路徑插到 `PATH` 前面，使該版軟體優先   |
| `pathJoin`                   | 安全拼接路徑                             |
| `setenv("HELLO_HOME", root)` | 設定軟體家目錄變數，供編譯／執行腳本使用 |
| `whatis(...)`                | `module whatis`／說明文字                |

- **意義**: 模組檔不「安裝軟體」，只描述「載入時環境要怎麼變」；軟體本體仍需事先安裝在 `root` 路徑。

### 學習心得：Lmod

- 叢集軟體管理的正確單位是「版本化環境」，不是大家共用同一組全域路徑。
- Lmod hierarchy 強迫較正確的載入順序，能減少難以除錯的 ABI／MPI 混用問題。
- 作業腳本（`sbatch`）內應明確 `ml add` 所需模組，確保非互動佇列環境可重現。

### 練習：Lmod

1. 比較 Environment Modules 與 Lmod 的至少三項差異。
2. 解釋 `$MODULEPATH` 與 `ml avail` 的關係。
3. 仿照範例寫一份 Lua modulefile，載入後需設定 `PATH` 與一個 `*_HOME` 變數。
4. 說明為何 `sbatch` 腳本裡也要 `module load`／`ml add`。

---

## 步驟總結

建置與使用 HPC 基礎能力，建議依下列順序：

1. **打好共用基礎**: SSH 金鑰、主機名解析、Chrony、（Slurm 用）Munge
2. **建立節點屬性**: 編寫 `/etc/genders`，用 `nodeattr` 驗證
3. **啟用平行控管**: 安裝 pdsh／pdsh-genders，先用無害指令測試目標集合
4. **部署感控制器**: 設定 `slurm.conf`、日誌目錄、防火牆，啟動 `slurmctld`
5. **部署計算節點**: 安裝 `slurm-node`、同步設定、啟動 `slurmd`，用 `sinfo` 確認
6. **跑通作業路徑**: `srun` → `sbatch` → `squeue`／`scancel`
7. **（可選）會計**: MariaDB → `slurmdbd.conf` → 改 `AccountingStorageType` → `sacct`／`sacctmgr`
8. **軟體環境**: 安裝 Lmod，用 `ml avail/add/list/purge` 管理版本，並在作業腳本中顯式載入

---

## 總結

本章建立 HPC 叢集管理的三條主線：以 **genders + pdsh** 解決「對誰下指令」，以 **Slurm** 解決「何時、用多少資源跑作業」，以 **Lmod** 解決「用哪一版軟體來跑」。佇列系統的歷史從 LoadLeveler、PBS、SGE、TORQUE 走到 Slurm，顯示資源管理已從簡單批次進化為可擴展、可回填、可記帳的政策引擎。實務上應先求控管與排程最小閉環可用，再疊加 slurmdbd 與複雜 module hierarchy；任何批次操作與排程設定，都以可驗證、可回復為前提，才能在多使用者共用環境中穩定運作。
