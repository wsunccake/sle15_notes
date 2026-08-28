# ch2. 練習題 — 路徑、權限、文字工具、YaST 與系統設定

依 `raw/ch2.md`、`raw/ch2-3.md` 整理的實作與問答練習。目標：理解 **Absolute Path / Relative Path**，讀懂權限與 `umask`／特殊權限；熟練 `cat`／`grep`／`head`／`tail`／`more`／`less`；會用 **YaST** 與指令列設定網路、防火牆與 **zypper**。

**建議環境**

- SLES 15（或相容 Linux）任一台可登入的系統
- 一般使用者帳號（範例使用者名：`alex`；實作時可改為目前登入使用者）
- 練習 4～5 部分步驟需要 `root` 或 `sudo`，以及可用的網路介面

---

## 練習 1. Absolute Path 與 Relative Path

### 1-1. 概念說明

1. 解釋什麼是 **Absolute Path（絕對路徑）**。
2. 解釋什麼是 **Relative Path（相對路徑）**。
3. 各舉一個例子，並說明兩者在腳本／自動化中的取捨。

### 1-2. 依目錄樹作答

使用者為 **alex**，目錄結構如下：

```text
/
├── home/
│   └── alex/
│       ├── documents/
│       │   └── report.txt
│       └── downloads/
│           └── image.png
└── var/
    └── log/
        └── sys.log
```

#### 1-2-1. Absolute Path

寫出下列檔案的 **Absolute Path**：

| 檔案         | Absolute Path |
| ------------ | ------------- |
| `report.txt` |               |
| `sys.log`    |               |

#### 1-2-2. Relative Path

假設目前工作目錄為 `/home/alex`，寫出：

| 檔案         | Relative Path（相對於 `/home/alex`） |
| ------------ | ------------------------------------ |
| `report.txt` |                                      |
| `sys.log`    |                                      |

#### 1-2-3. 用 Absolute Path 切換目錄

目前目錄：`/home/alex`  
請寫出以 **Absolute Path** 切換到 `log` 目錄（即 `/var/log`）的指令。

```bash
# 請填寫
cd ???
```

#### 1-2-4. 用 Relative Path 切換目錄

目前目錄：`/home/alex`  
請寫出以 **Relative Path** 切換到 `downloads` 的指令。

```bash
# 請填寫
cd ???
```

### 作答／驗收

1. 完成 1-1～1-2-4 文字作答。
2. （實作加分）在系統上建立類似目錄樹，用 `pwd`／`cd`／`ls` 驗證上述路徑。

---

## 練習 2. 檔案權限與 chmod

### 2-1. 解讀權限字串

給定權限：

```text
-rwxr-xr--
```

| 小題  | 問題                                          | 作答 |
| ----- | --------------------------------------------- | ---- |
| 2-1-1 | **Owner** 有哪些權限？（用 r／w／x 說明）     |      |
| 2-1-2 | **Group** 有哪些權限？                        |      |
| 2-1-3 | **Others** 有哪些權限？                       |      |
| 2-1-4 | 轉換成 **八進位數字權限**（如 `755`）是多少？ |      |

提示：`r=4`、`w=2`、`x=1`；字串最左側 `-` 表示一般檔案（directory 則為 `d`）。

### 2-2. 預設權限與 chmod

在家目錄（或練習目錄）執行：

```bash
touch test.txt
mkdir testdir
ls -l test.txt
ls -ld testdir
```

| 小題  | 任務                                                                               |
| ----- | ---------------------------------------------------------------------------------- |
| 2-2-1 | 記錄 `test.txt` 的**預設權限**（`ls -l` 字串與數字）                               |
| 2-2-2 | 記錄 `testdir` 的**預設權限**（`ls -ld` 字串與數字）                               |
| 2-2-3 | 將 `test.txt` 改為 `-rw-r--r--`，寫出使用的 `chmod` 指令（符號模式或數字模式皆可） |
| 2-2-4 | 將 `testdir` 改為 `drwxr-x---`，寫出使用的 `chmod` 指令，並用 `ls -ld` 驗證        |

**參考指令型態**

```bash
chmod 644 test.txt
# 或
chmod u=rw,g=r,o=r test.txt

chmod 750 testdir
# 或
chmod u=rwx,g=rx,o= testdir
```

### 2-3. 檔案與目錄上的 `r`、`x`

分別說明：

| 對象     | 權限 | 作用／意義 |
| -------- | ---- | ---------- |
| **檔案** | `r`  |            |
| **檔案** | `x`  |            |
| **目錄** | `r`  |            |
| **目錄** | `x`  |            |

補充思考題：

1. 目錄只有 `r`、沒有 `x` 時，`ls` 與 `cd` 可能出現什麼現象？
2. 檔案有 `r`、沒有 `x` 時，對「腳本」與「純文字」各代表什麼？

### 2-4. umask

執行並觀察：

```bash
umask 027
touch my.txt
mkdir mydir
ls -l my.txt
ls -ld mydir
```

| 小題  | 任務                                                                              |
| ----- | --------------------------------------------------------------------------------- |
| 2-4-1 | 說明 `umask` 的意義（與「建立檔案／目錄時的預設權限」關係）                       |
| 2-4-2 | 在 `umask 027` 下，一般檔案的理論預設權限如何計算？（可由 `666` 減去 umask 說明） |
| 2-4-3 | 在 `umask 027` 下，目錄的理論預設權限如何計算？（可由 `777` 減去 umask 說明）     |
| 2-4-4 | 記錄實際 `my.txt`、`mydir` 的權限，並與理論值對照；若有差異請說明可能原因         |

還原提示（實作後可恢復）：

```bash
umask 022
# 或登出後重新登入以還原 session 預設
```

### 2-5. 特殊權限：SUID、SGID、Sticky Bit

解釋下列特殊權限的**名稱、顯示符號、作用與常見場景**：

| 項目                     | 請說明                                                   |
| ------------------------ | -------------------------------------------------------- |
| **SUID**（Set User ID）  | 符號（如 `s`／`S`）、對執行檔的意義、一例（如 `passwd`） |
| **SGID**（Set Group ID） | 對執行檔／目錄的意義差異、一例                           |
| **Sticky Bit**           | 對目錄的意義、一例（如 `/tmp`）                          |

（加分）寫出設定範例指令：

```bash
chmod u+s <file>
chmod g+s <dir>
chmod +t /tmp
# 或數字模式（如 4755、2755、1777）並說明各位元意義
```

---

## 練習 3. 文字處理工具（cat / grep / head / tail / more / less）

### 3-1. 建立並查詢 `log.txt`

使用 `cat`（heredoc 或互動輸入）將下列內容寫入 `log.txt`：

```text
INFO Server started
INFO Loading config
ERROR Connection failed
WARNING CPU usage high
INFO User alice logged in
ERROR Authentication failed
INFO User bob logged in
WARNING Memory usage high
ERROR Connection timeout
INFO Server restarted
```

完成下列任務，並寫下使用的指令與結果摘要：

| 小題  | 任務                         | 建議指令方向                     |
| ----- | ---------------------------- | -------------------------------- |
| 3-1-1 | 將內容寫入 `log.txt`         | `cat > log.txt << 'EOF' ... EOF` |
| 3-1-2 | 顯示 `log.txt` **全部內容**  | `cat log.txt`                    |
| 3-1-3 | 顯示全部內容，並**顯示行號** | `cat -n log.txt` 或 `nl log.txt` |
| 3-1-4 | 只顯示**前 3 行**            | `head -n 3 log.txt`              |
| 3-1-5 | 只顯示**最後 3 行**          | `tail -n 3 log.txt`              |
| 3-1-6 | 找出所有含 **ERROR** 的行    | `grep ERROR log.txt`             |
| 3-1-7 | 找出所有**不是 INFO** 的行   | `grep -v INFO log.txt`           |

### 3-2. 持續顯示系統日誌

對系統日誌做**持續追蹤**（follow）：

```bash
# SLES 上常見路徑可能是 /var/log/messages；若存在 syslog 則用下列指令
sudo tail -f /var/log/messages
# 或
sudo tail -f /var/log/syslog
```

| 小題  | 任務                                                                               |
| ----- | ---------------------------------------------------------------------------------- |
| 3-2-1 | 寫出實際使用的檔案路徑與指令                                                       |
| 3-2-2 | 說明 `tail -f` 的意義，以及如何結束追蹤（通常 `Ctrl+C`）                           |
| 3-2-3 | （加分）另開終端產生一點日誌（如失敗登入／重載服務），觀察 follow 畫面是否出現新行 |

### 3-3. 過濾設定檔：去掉空白行與註解

針對 `/etc/ssh/ssh_config`（或系統上存在的同等檔），**不要顯示空白行與註解行**（以 `#` 開頭的註解為主）。

```bash
# 參考方向（可調整）
grep -vE '^\s*(#|$)' /etc/ssh/ssh_config
# 或分步：去掉註解、再去掉空行
```

| 小題  | 任務                                    |
| ----- | --------------------------------------- |
| 3-3-1 | 寫出完整指令                            |
| 3-3-2 | 附上過濾後輸出的前若干行作為證明        |
| 3-3-3 | 簡述 `grep -v` 與正規表示式在此題的作用 |

### 3-4. `more` 分頁瀏覽

```bash
more /etc/passwd
```

| 操作                  | 請寫出按鍵／作法 |
| --------------------- | ---------------- |
| 往下一頁              |                  |
| 往下一行              |                  |
| 離開 `more`（若適用） |                  |

### 3-5. `less` 瀏覽與搜尋

```bash
less /etc/passwd
```

| 操作         | 請寫出按鍵／指令 |
| ------------ | ---------------- |
| 往下移動     |                  |
| 往上移動     |                  |
| 回到檔案開頭 |                  |
| 跳到檔案結尾 |                  |
| 搜尋 `root`  |                  |
| 搜尋 `bash`  |                  |
| 離開 `less`  |                  |

---

## 練習 4. YaST（TUI / GUI）

在 SLES 上分別體驗文字與圖形介面的 YaST，並完成網路、防火牆與軟體相關設定（實驗室環境可操作；正式機請謹慎）。

| 小題 | 任務                                                                | 參考                                                |
| ---- | ------------------------------------------------------------------- | --------------------------------------------------- |
| 4-1  | 在 **TUI** 使用 YaST                                                | `sudo yast` 或 `sudo yast2`（文字模式）             |
| 4-2  | 在 **GUI** 使用 YaST                                                | 圖形桌面啟動 YaST，或具 X11／遠端桌面時執行 `yast2` |
| 4-3  | 使用 YaST 設定 **Network（網路）** 與 **Firewall（防火牆）**        | 模組如 `yast lan`、`yast firewall`                  |
| 4-4  | 使用 YaST 設定 **Repository（軟體庫）** 與 **Software（安裝軟體）** | 模組如 `yast repositories`、`yast sw_single`        |

---

## 練習 5. 指令列：網路、防火牆、zypper

> 操作前建議備份設定檔；靜態 IP 請使用實驗室網段、避免位址衝突。

### 5-1. 網路介面（ifcfg + wicked）

編輯 `/etc/sysconfig/network/ifcfg-<interface>`（將 `<interface>` 換成實際名稱，如 `eth0`）。

| 小題  | 任務                                                                                |
| ----- | ----------------------------------------------------------------------------------- |
| 5-1-1 | 使用 `vi`（或 `vim`）設定 **Static IP**（含 `BOOTPROTO`、`IPADDR`、`STARTMODE` 等） |
| 5-1-2 | 使用 `vi` 設定 **DHCP**（`BOOTPROTO='dhcp'` 等）                                    |
| 5-1-3 | 使用 `wicked` 對介面執行 **up / down / show**，並記錄結果                           |

**參考片段**

```ini
# Static 概念
BOOTPROTO='static'
STARTMODE='auto'
IPADDR='192.168.x.y/24'
GATEWAY='192.168.x.1'

# DHCP 概念
BOOTPROTO='dhcp'
STARTMODE='auto'
```

```bash
sudo wicked ifdown <interface>
sudo wicked ifup <interface>
sudo wicked show <interface>
# 或 ifreload
sudo wicked ifreload <interface>
```

### 5-2. firewall-cmd

| 小題  | 任務                            | 參考型態                                          |
| ----- | ------------------------------- | ------------------------------------------------- |
| 5-2-1 | **add service**（永久＋reload） | `firewall-cmd --permanent --add-service=ssh`      |
| 5-2-2 | **del service**                 | `firewall-cmd --permanent --remove-service=...`   |
| 5-2-3 | **add port**                    | `firewall-cmd --permanent --add-port=8080/tcp`    |
| 5-2-4 | **del port**                    | `firewall-cmd --permanent --remove-port=8080/tcp` |

每次永久變更後執行：

```bash
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

### 5-3. zypper 軟體庫與套件

| 小題  | 任務                | 參考型態                                      |
| ----- | ------------------- | --------------------------------------------- |
| 5-3-1 | **add repo**        | `zypper ar <URI> <alias>` 或 `zypper addrepo` |
| 5-3-2 | **del repo**        | `zypper rr <alias>` 或 `zypper removerepo`    |
| 5-3-3 | **search package**  | `zypper se <keyword>`                         |
| 5-3-4 | **install package** | `zypper in <package>`                         |
| 5-3-5 | **remove package**  | `zypper rm <package>`                         |

驗證常用：

```bash
sudo zypper lr -d
sudo zypper se vim
rpm -q <package>
```
