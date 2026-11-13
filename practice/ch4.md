# ch4. 練習題 — Service：systemd 與網路服務 Client／Server 架設

依 `content/ch4.md` 整理的實作與問答練習。目標：先掌握 **systemd**（unit／target／生命週期），再依「安裝 → 設定 → 語法檢查 → `enable --now` → 防火牆 → **從 Client 驗證**」完成各服務的 **Server／Client 架設**。

**建議環境**

- SLES 15（或相容 Linux）**兩台**（沿用 ch1 的 VM-A／VM-B 即可）
- 一台當 **Server**、一台當 **Client**，同網段、可互 ping
- 多數步驟需要 `root` 或 `sudo`
- **DHCP／PXE／Telnet／TFTP** 僅在隔離實驗網操作；勿在公司／宿舍正式網段啟用第二台 DHCP
- NIS 安全模型偏舊，僅供理解目錄服務古典模型，完成後可停用

**建議主機命名與位址（可依實驗室調整，但報告必須寫實際值）**

| 角色     | hostname 建議  | IP 建議            | 備註                         |
| -------- | -------------- | ------------------ | ---------------------------- |
| Server   | `sle15-server` | `192.168.1.10/24`  | 靜態；提供各項服務           |
| Client   | `sle15-client` | `192.168.1.20/24`  | 先靜態；DHCP 練習再改 dhcp   |
| 網段     | —              | `192.168.1.0/24`   | NIS domain 建議：`sle15domain` |
| DNS zone | —              | `example.com`      | 正向／反向各一區             |

開題前先填：

| 項目              | Server           | Client           |
| ----------------- | ---------------- | ---------------- |
| 實際 hostname     |                  |                  |
| 實際 IP／CIDR     |                  |                  |
| 閘道              |                  |                  |
| 兩機 `ping` 結果  |                  |                  |

**各服務固定檢查清單（每做完一項就打勾）**

1. 套件是否安裝
2. 設定檔是否正確、有無語法檢查
3. `systemctl status` 是否 `active`
4. `firewall-cmd --list-services`／ports 是否放行
5. **Client 端功能驗證是否成功**

---

## 練習 0. 實驗拓樸與通用操作模式

### 0-1. 角色與互通

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 0-1-1 | 指定哪台是 Server、哪台是 Client，寫下 hostname 與 IP                |
| 0-1-2 | Server `ping` Client、Client `ping` Server，記錄是否通、RTT          |
| 0-1-3 | 說明後續為什麼幾乎每個服務都要「本機 active」加上「Client 測一次」   |

### 0-2. 通用六步（多數服務相同）

用自己的話填完，後面各練習只寫差異：

| 步驟 | 通用作法                         | 你會用的代表指令 |
| ---- | -------------------------------- | ---------------- |
| 1    | 安裝套件                         |                  |
| 2    | 編輯設定檔（必要時先備份）       |                  |
| 3    | 檢查語法（若工具支援）           |                  |
| 4    | 開機啟用並立即啟動               |                  |
| 5    | 防火牆永久放行後 reload          |                  |
| 6    | 從 Client 驗證功能               |                  |

### 作答／驗收（練習 0）

1. 兩機互通證明（ping 或 SSH）。
2. 能默寫通用六步，且知道第 6 步不能省略。

---

## 練習 1. systemd：服務與開機管理

幾乎所有後續服務（`sshd`、`chronyd`、`named`、`apache2` 等）都透過 systemd 管理。先把單機生命週期做熟，再做跨機架設。

### 1-1. 演進與 runlevel／target

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 1-1-1 | 以表格對照 **SysVinit**、**Upstart**、**systemd** 至少三項差異       |      |
| 1-1-2 | SysV **runlevel 3**／**5** 分別對應哪個 systemd **target**？         |      |
| 1-1-3 | runlevel 0／1／6 常見對應哪些 target？                               |      |
| 1-1-4 | 查目前預設 target、改為 `multi-user.target` 的指令各是什麼？         |      |
| 1-1-5 | 想在命令列切到文字多使用者模式，應使用哪道指令？`isolate` 有何風險？ |      |
| 1-1-6 | 舊文件寫 `service foo start` 或 `/etc/init.d/foo`，在 SLES 15 多半對應什麼？ | |

```bash
systemctl get-default
systemctl set-default multi-user.target
runlevel
systemctl isolate multi-user.target
```

### 1-2. 生命週期指令

對現有服務（建議 `sshd`）操作並記錄：

```bash
systemctl status sshd
systemctl start sshd
systemctl stop sshd
systemctl restart sshd
systemctl reload sshd
systemctl enable sshd
systemctl disable sshd
systemctl enable --now sshd
systemctl is-active sshd
systemctl is-enabled sshd
```

| 指令              | 意義（請填） | 是否改「開機自動啟動」 |
| ----------------- | ------------ | ---------------------- |
| `start`／`stop`   |              |                        |
| `restart`         |              |                        |
| `reload`          |              |                        |
| `enable`／`disable` |            |                        |
| `enable --now`    |              |                        |

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 1-2-1 | `list-units` 與 `list-unit-files` 差別？                             |      |
| 1-2-2 | 改 unit 檔後為什麼要 `daemon-reload`？忘記做會看到什麼現象？         |      |
| 1-2-3 | 寫出「立即啟動 + 開機啟用 + 看日誌」的指令序列                       |      |
| 1-2-4 | 日誌為何優先 `journalctl -u <svc>`，再查服務自己的 log 檔？          |      |

```bash
systemctl cat sshd.service
systemctl daemon-reload
journalctl -u sshd -xe
journalctl -u sshd -f
```

### 1-3. 自訂 unit（單機「服務端」部署）

撰寫 `/etc/systemd/system/myapp.service`（`Type=simple`），用簡單迴圈模擬常駐行程。

```bash
mkdir -p /opt/myapp/bin
cat > /opt/myapp/bin/myapp.sh << 'EOF'
#!/bin/bash
while true; do
  echo "$(date) myapp running"
  sleep 5
done
EOF
chmod 755 /opt/myapp/bin/myapp.sh
```

| 區段／參數                         | 意義（請填） |
| ---------------------------------- | ------------ |
| `[Unit] Description`               |              |
| `After=`                           |              |
| `Wants=`／`Requires=`              |              |
| `[Service] Type=simple`            |              |
| `ExecStart=`                       |              |
| `Restart=`                         |              |
| `[Install] WantedBy=`              |              |
| `/usr/lib/systemd/system/` vs `/etc/systemd/system/` | |
| drop-in（`systemctl edit`）        |              |

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 1-3-1 | 寫出完整 `myapp.service`（含 `After`、`ExecStart`、`WantedBy`）      |
| 1-3-2 | 執行 `daemon-reload` → `enable --now` → `status` → `journalctl -u myapp` |
| 1-3-3 | 說明為何修改套件原廠 unit 時優先用 drop-in，而不是直接改 `/usr/lib/...` |
| 1-3-4 | 實驗結束後 `disable --now myapp`（可選刪檔）                         |

### 作答／驗收（練習 1）

1. 完成 1-1 對照表與 target 對應。
2. 附上 `systemctl status sshd` 與 `is-enabled` 結果。
3. 自訂 `myapp.service` 能 `active`，journal 看得到迴圈輸出。

---

## 練習 2. SSH：Client／Server 架設

遠端管理底座。先讓 Client 能登入 Server，再談強化參數。任何會鎖死登入的變更，都要保留一個已登入的 session。

### 2-1. 協定演進

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 2-1-1 | 比較 `rsh`／`rlogin`、`Telnet`、`SSH` 在「傳輸是否加密」上的差異     |      |
| 2-1-2 | SSH 的加密、多種驗證、完整性保護、擴充（SFTP／埠轉發）各代表什麼？   |      |
| 2-1-3 | 為什麼明文協定不該再用於管理面？                                     |      |

### 2-2. Server 架設

在 **Server** 完成：

```bash
server:~ # zypper install openssh
server:~ # systemctl enable --now sshd
server:~ # systemctl status sshd
server:~ # firewall-cmd --permanent --add-service=ssh
server:~ # firewall-cmd --reload
server:~ # firewall-cmd --list-services
server:~ # cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

| 小題  | 任務                                                                 | 作答／結果 |
| ----- | -------------------------------------------------------------------- | ---------- |
| 2-2-1 | 安裝並使 `sshd` 開機啟用且立即 `active`                              |            |
| 2-2-2 | 防火牆永久放行 `ssh` 並 reload，記錄 `list-services`                 |            |
| 2-2-3 | 說明為何改 `sshd_config` 前要備份，且改完要 `sshd -t` 再 `reload`    |            |

### 2-3. Client 連線驗證

```bash
client:~ # ssh <user>@192.168.1.10
client:~ # ssh -p 2222 <user>@192.168.1.10
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 2-3-1 | 從 Client 以密碼（或既有金鑰）登入 Server，記錄指令與成功證明        |
| 2-3-2 | （建議）Client 產生金鑰並把公鑰放到 Server 的 `authorized_keys`      |

**金鑰參考方向**

```bash
client:~ # ssh-keygen -t rsa -b 4096
client:~ # ssh-copy-id <user>@192.168.1.10
# 若已改埠：
client:~ # ssh-copy-id -p 2222 <user>@192.168.1.10
```

### 2-4. `sshd_config` 強化（實驗機；先確認金鑰可登入）

| 參數 | 用途與意義（請填） |
| ---- | ------------------ |
| `Port` |                    |
| `ListenAddress` |            |
| `PermitRootLogin` |          |
| `PasswordAuthentication` | |
| `PubkeyAuthentication` |   |
| `PermitEmptyPasswords` |   |
| `X11Forwarding` |            |
| `MaxAuthTries` |             |
| `AllowUsers`／`AllowGroups` | |
| `LoginGraceTime` |           |

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 2-4-1 | 寫出禁止 root 密碼直登、改用金鑰的建議參數組合                       |      |
| 2-4-2 | 關閉密碼前必須先確認什麼？為什麼要留備援 session？                   |      |
| 2-4-3 | 若改 `Port 2222`，列出必須同步修改的項目（Client 連線方式、防火牆等） |    |
| 2-4-4 | 改非標準埠為什麼只是「模糊安全」，不能取代金鑰與權限控管？           |      |

```bash
server:~ # sshd -t
server:~ # systemctl reload sshd
server:~ # firewall-cmd --permanent --add-port=2222/tcp
server:~ # firewall-cmd --reload
```

安全層次（請用自己的話排序／補充）：網路隔離 → 防火牆 → 非 root 直登 → 金鑰 → 使用者白名單。

### 作答／驗收（練習 2）

1. Server：`sshd` active、防火牆已放行。
2. Client：成功 `ssh` 登入（附指令與提示字元／hostname）。
3. 完成強化參數表；若有改埠，Client 必須用 `-p` 再連一次。

---

## 練習 3. Chrony：內網時間 Server／Client 架設

憑證、日誌關聯、Kerberos、叢集、MUNGE 都依賴正確時間。SLES 常以 **chronyd** 為預設。

### 3-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 3-1-1 | 比較 NTP、SNTP、Chrony 的適用情境（含 VM／不穩網路）                 |      |
| 3-1-2 | Stratum 分層與「多伺服器冗餘」在防什麼？                             |      |
| 3-1-3 | `iburst` 與 `makestep 1.0 3` 各自解決什麼問題？                      |      |
| 3-1-4 | `driftfile`、`rtcsync`、`allow`、`local stratum 10` 各做什麼？       |      |
| 3-1-5 | 用 `chronyc tracking` 如何判讀是否已同步？                           |      |

### 3-2. Server 架設（對內網提供 NTP）

```bash
server:~ # zypper install chrony
server:~ # vi /etc/chrony.conf
# allow 192.168.1.0/24
# local stratum 10
server:~ # systemctl enable --now chronyd
server:~ # firewall-cmd --permanent --add-service=ntp
# 或 UDP/123
server:~ # firewall-cmd --reload
server:~ # chronyc sources
server:~ # chronyc tracking
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 3-2-1 | 安裝並啟用 `chronyd`，記錄 `tracking` 摘要                           |
| 3-2-2 | 設定 `allow` 僅內網、必要時 `local stratum 10`（上游全掛時仍能對內服務） |
| 3-2-3 | 防火牆放行 NTP（UDP/123），說明若本機當 Server 卻沒 `allow`／沒開埠會怎樣 |

### 3-3. Client 架設與驗證

Client 指向上一步的 Server IP（先註解或停用公共 pool，避免分不清來源）：

```bash
client:~ # zypper install chrony
client:~ # vi /etc/chrony.conf
# server 192.168.1.10 iburst
client:~ # systemctl enable --now chronyd
client:~ # chronyc sources
client:~ # chronyc tracking
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 3-3-1 | Client 的時間源是否顯示為 Server IP？附 `chronyc sources`            |
| 3-3-2 | 從 `tracking` 判斷是否同步（stratum、offset 概念）                   |
| 3-3-3 | 說明 VM 環境為何更應使用 Chrony，並確認可達上游或內網 NTP            |

### 作答／驗收（練習 3）

1. Server 能對 `192.168.1.0/24` 提供時間。
2. Client `sources` 指向該 Server，且已同步（或說明實驗網無法出外時 `local stratum` 的現象）。

---

## 練習 4. NIS：集中帳號 Client／Server 架設

NIS（原 Yellow Pages）可在內網集中發佈帳號／群組。現代企業更常 LDAP／AD；本題練習古典目錄模型。NIS domain **不是** DNS domain。

### 4-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 4-1-1 | 說明 `ypinit -m` 與後續 `ypmake`／`make` 的時機與差異                |      |
| 4-1-2 | 設計一組只允許 `192.168.1.0/24` 的 `/etc/ypserv.conf` 規則（含最後 deny） | |
| 4-1-3 | `/var/yp/securenets` 用途是什麼？                                    |      |
| 4-1-4 | 為什麼帳號一改，Server 必須重建 map，Client 才看得到？               |      |

### 4-2. Server 架設

```bash
server:~ # zypper install ypserv yast2-nis-server
server:~ # ypdomainname sle15domain
server:~ # ypdomainname > /etc/defaultdomain
server:~ # /usr/lib/yp/ypinit -m
server:~ # vi /etc/ypserv.conf
server:~ # vi /var/yp/securenets
server:~ # firewall-cmd --permanent --add-service=rpc-bind
server:~ # firewall-cmd --reload
server:~ # systemctl enable --now ypserv
server:~ # /usr/lib/yp/ypmake
# 或：cd /var/yp && make
```

可搭配 YaST：`yast nis_server`。

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 4-2-1 | 設定 NIS domain（寫入 `defaultdomain`），初始化 master 資料庫        |
| 4-2-2 | 寫出 `ypserv.conf` 允許本機 + `192.168.1.0/24`、其餘 deny            |
| 4-2-3 | 在 Server 建一個測試帳號後執行 `ypmake`，說明為何要重建              |
| 4-2-4 | `ypserv` 與 `rpcbind` 皆為 `active`                                  |

### 4-3. Client 架設與驗證

```bash
client:~ # zypper install ypbind yast2-nis-client
client:~ # ypdomainname sle15domain
client:~ # vi /etc/yp.conf
# domain sle15domain server 192.168.1.10
client:~ # vi /etc/nsswitch.conf
# passwd / group 等加入 nis；並留意 PAM
client:~ # firewall-cmd --permanent --add-service=rpc-bind
client:~ # firewall-cmd --reload
client:~ # systemctl enable --now ypbind
client:~ # ypdomainname
client:~ # ypwhich
client:~ # getent passwd
```

可搭配 YaST：`yast nis`。

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 4-3-1 | Client domain 與 Server 一致，`yp.conf` 指向 Server IP               |
| 4-3-2 | `ypwhich` 顯示綁定哪台 NIS server                                    |
| 4-3-3 | `getent passwd` 能看到 Server 上的 NIS 帳號                          |
| 4-3-4 | 若綁定失敗，先查 NIS domain 名稱是否與 DNS domain 搞混               |

### 作答／驗收（練習 4）

1. Server：`ypinit -m` 完成、`ypserv` active、網段存取控制已寫。
2. Client：`ypwhich` + `getent passwd` 證明已綁定。
3. 附上 `ypserv.conf` 規則（可遮罩真實網段以外的敏感資訊）。

---

## 練習 5. NFS：網路檔案共用 Client／Server 架設

Client 把遠端目錄掛成本地路徑。權限由「匯出選項 + Server 本地 POSIX」共同決定。正式環境勿隨意 `chmod 777`。

### 5-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 5-1-1 | 解釋 `root_squash` 的安全意義；`no_root_squash` 有何風險？           |      |
| 5-1-2 | 寫一條僅允許 `192.168.1.0/24` **唯讀** 的 `/etc/exports`             |      |
| 5-1-3 | `rw`／`ro`、`sync`／`async`、`no_subtree_check`、`all_squash` 各做什麼？ | |
| 5-1-4 | `exportfs -a`／`-r`／`-v` 差別？                                     |      |
| 5-1-5 | 說明 `_netdev` 為何對開機掛載很重要                                  |      |

### 5-2. Server 架設

```bash
server:~ # zypper install nfs-kernel-server rpcbind yast2-nfs-server
server:~ # mkdir -p /srv/nfs/shared_data
server:~ # chmod 755 /srv/nfs/shared_data
server:~ # echo test-nfs > /srv/nfs/shared_data/hello.txt
server:~ # vi /etc/exports
# /srv/nfs/shared_data 192.168.1.0/24(rw,sync,no_subtree_check)
server:~ # exportfs -a
server:~ # systemctl enable --now rpcbind
server:~ # systemctl enable --now nfs-server
server:~ # firewall-cmd --permanent --add-service=rpc-bind
server:~ # firewall-cmd --permanent --add-service=nfs
server:~ # firewall-cmd --permanent --add-service=mountd
server:~ # firewall-cmd --permanent --add-service=nfs3
server:~ # firewall-cmd --reload
```

可搭配 YaST：`yast nfs_server`。

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 5-2-1 | 建立共享目錄、寫入測試檔、設定 exports（先可用 `rw` 方便驗證寫入）   |
| 5-2-2 | `nfs-server`／`rpcbind` active，四項相關 firewall service 已放行     |
| 5-2-3 | 說明 NFSv3 為何常需額外 `mountd`／相關埠                             |

### 5-3. Client 架設與驗證

```bash
client:~ # zypper install nfs-client rpcbind
client:~ # mkdir -p /mnt/nfs_share
client:~ # mount -t nfs 192.168.1.10:/srv/nfs/shared_data /mnt/nfs_share
client:~ # ls /mnt/nfs_share
client:~ # vi /etc/fstab
# 192.168.1.10:/srv/nfs/shared_data /mnt/nfs_share nfs defaults,noatime,_netdev 0 0
```

可搭配 YaST：`yast nfs`。

| fstab 欄位／選項 | 意義（請填） |
| ---------------- | ------------ |
| 遠端路徑         |              |
| 本地掛載點       |              |
| `nfs`            |              |
| `defaults`       |              |
| `noatime`        |              |
| `_netdev`        |              |
| `0 0`            |              |

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 5-3-1 | 手動 `mount` 成功，能讀到 `hello.txt`                                |
| 5-3-2 | （若 exports 為 rw）在 Client 建立一個檔，回 Server 目錄核對         |
| 5-3-3 | 寫入 fstab（含 `_netdev`）；說明如何驗證開機自動掛載                 |
| 5-3-4 | 另寫一條 **唯讀** exports 規則（文字作答即可，或另開測試目錄）       |

### 作答／驗收（練習 5）

1. Server exports 與防火牆完成。
2. Client 掛載成功，附 `mount`／`ls` 輸出。
3. 能解釋 `root_squash` 與 `_netdev`。

---

## 練習 6. DNS：BIND（named）Client／Server 架設

DNS 角色分 **Authoritative Server**（權威回答某 zone）與 **Recursive Resolver**（代客查詢）。本練習在 Server 上架 **BIND**，Client 指定該 nameserver 查正向 A 與反向 PTR。

### 6-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 6-1-1 | 比較 Authoritative 與 Recursive 角色                                 |      |
| 6-1-2 | BIND、PowerDNS、Unbound、dnsmasq、CoreDNS 各適合什麼場景？（一句即可） | |
| 6-1-3 | `named.conf` 裡 `options`、正向 zone、反向 zone、hint 各做什麼？     |      |
| 6-1-4 | 新增 `pc2 IN A 192.168.1.12` 時，SOA **serial** 應如何處理？為什麼？ |      |
| 6-1-5 | SLES 上 `resolv.conf` 可能被誰覆寫？應一併留意哪個 ifcfg／服務？     |      |

### 6-2. Server 架設

建議記錄（可改成你的真實 IP）：

| 名稱                    | 值建議                |
| ----------------------- | --------------------- |
| zone                    | `example.com`         |
| NS／Server              | `s1.example.com` → `192.168.1.10` |
| 測試主機                | `pc1.example.com` → `192.168.1.20`（Client） |
| 反向 zone               | 依網段，如 `1.168.192.in-addr.arpa` |

```bash
server:~ # zypper in bind yast2-dns-server
server:~ # vi /etc/named.conf
server:~ # vi /var/lib/named/master/example.com
server:~ # vi /var/lib/named/master/1.168.192.in-addr.arpa
server:~ # named-checkconf
server:~ # named-checkzone example.com /var/lib/named/master/example.com
server:~ # named-checkzone 1.168.192.in-addr.arpa /var/lib/named/master/1.168.192.in-addr.arpa
server:~ # systemctl enable --now named
server:~ # firewall-cmd --permanent --add-service=dns
server:~ # firewall-cmd --reload
```

可搭配 YaST：`yast dns-server`。

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 6-2-1 | 完成正向 zone（至少 NS + 一筆 A）與對應 PTR                          |
| 6-2-2 | `named-checkconf` 與 `named-checkzone` 通過後再 enable               |
| 6-2-3 | `named` active，防火牆 `dns` 已放行                                  |
| 6-2-4 | 記錄 SOA serial；之後每改記錄就遞增一次                              |

正向／反向概念（請改成實際資料，勿照抄未改 IP）：

```conf
# 正向概念
$TTL 2D
@ IN SOA s1.example.com. root.s1.example.com. (
  2025081900 ; serial
  3H 1H 1W 1D )
@ IN NS s1.example.com.
s1 IN A 192.168.1.10
pc1 IN A 192.168.1.20

# 反向概念
20 IN PTR pc1.example.com.
```

### 6-3. Client 架設與驗證

```bash
client:~ # vi /etc/resolv.conf
# nameserver 192.168.1.10
# search example.com
# 或寫入 ifcfg，讓 wicked 管理
client:~ # vi /etc/sysconfig/network/ifcfg-eth0
# DNSCLIENT_DNS1='192.168.1.10'
client:~ # systemctl restart wicked
client:~ # dig @192.168.1.10 pc1.example.com
client:~ # dig -x 192.168.1.20 @192.168.1.10
client:~ # nslookup pc1.example.com
```

| 驗證               | 意義（請填） | 你的結果 |
| ------------------ | ------------ | -------- |
| `dig @server name` |              |          |
| `dig -x IP`        |              |          |
| `nslookup`         |              |          |

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 6-3-1 | Client 指定 Server 為 nameserver                                     |
| 6-3-2 | 用 `dig` **分別**驗證 A 與 PTR，附 ANSWER SECTION 摘要               |
| 6-3-3 | 說明為何驗證時常用 `dig @伺服器`，而不是只靠系統預設 resolver        |

### 作答／驗收（練習 6）

1. Server zone 檢查通過、`named` active。
2. Client 正向、反向查詢皆成功。
3. 能說明改記錄必改 serial。

---

## 練習 7. DHCP：位址分配 Client／Server 架設

同網段只能有一個「權威」位址池策略。請在 **隔離虛擬網** 操作，避免與家用／公司 DHCP 搶答。

### 7-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 7-1-1 | 解釋 `default-lease-time`／`max-lease-time` 對位址回收的影響         |      |
| 7-1-2 | 寫出以 MAC 綁固定 IP 的 `host` 段落（用 Client 真實 MAC）            |      |
| 7-1-3 | 說明為何要設定 `DHCPD_INTERFACE`；綁錯介面會有什麼現象？             |      |
| 7-1-4 | `option routers` 與 `option domain-name-servers` 發給 Client 什麼？  |      |
| 7-1-5 | 固定 IP「host 保留」與 Client 本機靜態設定，維運上如何取捨？         |      |

### 7-2. Server 架設

```bash
server:~ # zypper install dhcp-server
server:~ # vi /etc/dhcpd.conf
server:~ # vi /etc/sysconfig/dhcpd
# DHCPD_INTERFACE="eth0"    # 換成實際介面
server:~ # dhcpd -t -cf /etc/dhcpd.conf
server:~ # systemctl enable --now dhcpd
server:~ # firewall-cmd --permanent --add-service=dhcp
server:~ # firewall-cmd --reload
```

`dhcpd.conf` 概念（請改成實驗網段，勿與 Server 靜態 IP 衝突）：

```conf
option domain-name "example.com";
option domain-name-servers 192.168.1.10;
option routers 192.168.1.10;
default-lease-time 14400;
ddns-update-style none;

subnet 192.168.1.0 netmask 255.255.255.0 {
  range 192.168.1.101 192.168.1.150;
  default-lease-time 14400;
  max-lease-time 172800;
  host sle15-client {
    hardware ethernet XX:XX:XX:XX:XX:XX;
    fixed-address 192.168.1.20;
  }
}
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 7-2-1 | 寫出 subnet、range、routers、DNS，且 **range 不含 Server 自身 IP**   |
| 7-2-2 | 用 Client MAC 做 `host` 保留                                         |
| 7-2-3 | `dhcpd -t` 通過後再啟動；記錄 `DHCPD_INTERFACE`                      |
| 7-2-4 | 防火牆放行 `dhcp`                                                    |

### 7-3. Client 架設與驗證

先記錄 Client 目前靜態 IP，以便失敗時還原。

```bash
client:~ # vi /etc/sysconfig/network/ifcfg-eth0
# BOOTPROTO='dhcp'
# STARTMODE='auto'
client:~ # systemctl restart wicked
client:~ # ip addr show
client:~ # ip route
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 7-3-1 | Client 改 DHCP，取得位址；記錄 IP、mask、gateway、DNS                |
| 7-3-2 | 若使用 host 保留，確認拿到的是預期固定 IP                            |
| 7-3-3 | 若「Server 服務有起來但 Client 沒拿到 IP」，列出你會檢查的三件事（介面、防火牆、同網段是否有另一台 DHCP） |
| 7-3-4 | 本練習結束後，視需要把 Client 改回靜態，以免後續題目 IP 漂移         |

### 作答／驗收（練習 7）

1. 附上 `dhcpd -t` 成功與 `dhcpd` active。
2. Client `ip a` 證明有租約（或保留 IP）。
3. 報告註明實驗網隔離方式。

---

## 練習 8. Apache：Web Client／Server 架設

先讓 HTTP 通，再談 VirtualHost、TLS。每次重載前固定做 `apachectl configtest`。

### 8-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 8-1-1 | 說明為何修改設定後要先 `apachectl configtest`                        |      |
| 8-1-2 | `apache2-prefork` 是什麼？`httpd.conf` 與 conf.d／vhosts 關係？      |      |
| 8-1-3 | 防火牆為什麼常要同時考慮 `http`（80）與 `https`（443）？             |      |

### 8-2. Server 架設

```bash
server:~ # zypper install apache2 apache2-prefork yast2-http-server
server:~ # echo "hello from sle15-server" > /srv/www/htdocs/index.html
# 實際 DocumentRoot 若不同，請依 apache2 設定調整並在報告註明
server:~ # apachectl configtest
server:~ # systemctl enable --now apache2
server:~ # firewall-cmd --permanent --add-service=http
server:~ # firewall-cmd --permanent --add-service=https
server:~ # firewall-cmd --reload
```

可搭配 YaST：`yast http-server`。

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 8-2-1 | 安裝、`configtest` 通過、`apache2` active                            |
| 8-2-2 | 放行 http（實驗若無憑證，https 可只放行不強制測 TLS）                |
| 8-2-3 | 在 DocumentRoot 放一個可辨識的測試頁                                 |

### 8-3. Client 驗證

```bash
client:~ # curl -I http://192.168.1.10/
client:~ # curl http://192.168.1.10/
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 8-3-1 | 用 `curl -I` 記錄 HTTP 狀態碼                                        |
| 8-3-2 | 用 `curl` 取得頁面內容，確認是 Server 上放的測試字串                 |
| 8-3-3 | （加分）本機瀏覽器或第二台 Client 再開一次，結果應一致               |

### 作答／驗收（練習 8）

1. Server：`configtest` + `apache2` active + 防火牆。
2. Client：`curl -I` 與本文內容。

---

## 練習 9. FTP（vsftpd）：Client／Server 架設

FTP 控制與資料通道分離、且常明文。正式環境應優先 SFTP／FTPS；實驗室用來理解主動／被動模式與防火牆。

### 9-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 9-1-1 | 為何設定 `pasv_min_port`～`pasv_max_port` 後，防火牆也要開同樣範圍？ |      |
| 9-1-2 | 比較匿名唯讀與本機使用者登入的設定差異（`anonymous_enable`／`local_enable`／`write_enable`／`chroot_local_user`） | |
| 9-1-3 | 「能登入不能傳檔」最常見原因是什麼？                                 |      |
| 9-1-4 | 匿名寫入加上 `777` 目錄為什麼風險極高？                              |      |

### 9-2. Server 架設

```bash
server:~ # zypper install vsftpd yast2-ftp-server
server:~ # vi /etc/vsftpd.conf
# pasv_enable=YES
# pasv_min_port=40000
# pasv_max_port=40100
server:~ # systemctl enable --now vsftpd
server:~ # firewall-cmd --permanent --add-port=21/tcp
server:~ # firewall-cmd --permanent --add-port=40000-40100/tcp
server:~ # firewall-cmd --reload
```

可搭配 YaST：`yast ftp-server`。

| 參數 | 意義（請填） | 你的設定 |
| ---- | ------------ | -------- |
| `anonymous_enable` |          |          |
| `local_enable` |              |          |
| `write_enable` |              |          |
| `chroot_local_user` |         |          |
| `pasv_min_port`／`pasv_max_port` | |    |

### 9-3. Client 驗證

```bash
client:~ # ftp 192.168.1.10
```

| 命令          | 意義（請填） | 你是否實際用過 |
| ------------- | ------------ | -------------- |
| `ls`／`dir`   |              |                |
| `cd`／`lcd`   |              |                |
| `get`／`put`  |              |                |
| `binary`／`ascii` |          |                |
| `bye`         |              |                |

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 9-3-1 | Client 登入 Server FTP（匿名或本機測試帳號，報告註明）               |
| 9-3-2 | 完成至少一次 `ls` 與一次 `get`（或 `put`，若允許寫入）               |
| 9-3-3 | 若被動模式失敗，對照防火牆埠範圍與 `vsftpd.conf`                     |

### 作答／驗收（練習 9）

1. Server：`vsftpd` active，21 與被動埠範圍已開。
2. Client：能列目錄；傳檔成功或寫出失敗原因與修正。

---

## 練習 10. TELNET：對照學習（隔離網）

Telnet 明文傳輸，**僅**建議在隔離實驗網理解「為何被 SSH 取代」。做完立即停用。

### 10-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 10-1-1 | 用概念（或實驗網 `tcpdump`）說明 Telnet 登入為何危險                 |      |
| 10-1-2 | 「能通」為什麼仍不能用在管理面？                                     |      |

### 10-2. 短暫架設與 Client 連線

```bash
server:~ # zypper in telnet-server
server:~ # systemctl enable --now telnet
server:~ # firewall-cmd --permanent --add-service=telnet
server:~ # firewall-cmd --reload

client:~ # zypper in telnet
client:~ # telnet 192.168.1.10
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 10-2-1 | Client 連上後立即登出                                                |
| 10-2-2 | **停用**：`systemctl disable --now telnet`，並從防火牆移除 telnet    |

### 作答／驗收（練習 10）

1. 能說明明文風險。
2. 必須附上停用服務與關閉防火牆放行的指令；最終 `is-active` 應為 inactive。

---

## 練習 11. TFTP：開機檔／組態傳送 Client／Server 架設

TFTP 基於 UDP、無強認證，常用於 PXE。必須限制在管理網段。

### 11-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 11-1-1 | 說明 TFTP 與 FTP／SFTP 的安全差異                                    |      |
| 11-1-2 | 目錄權限與 `tftp` 使用者不正確時，常見症狀是什麼？                   |      |
| 11-1-3 | TFTP 在 PXE 鏈裡負責送什麼？                                         |      |

### 11-2. Server 架設

```bash
server:~ # zypper in tftp yast2-tftp-server
server:~ # vi /etc/sysconfig/tftp
server:~ # systemctl enable --now tftp
server:~ # firewall-cmd --permanent --add-service=tftp
server:~ # firewall-cmd --reload
server:~ # hostname > /srv/tftpboot/hostname
server:~ # chown -R tftp:tftp /srv/tftpboot
server:~ # chmod -R 755 /srv/tftpboot
```

可搭配 YaST：`yast tftp-server`。實際 tftp 根目錄若不是 `/srv/tftpboot`，以設定檔為準並在報告註明。

### 11-3. Client 驗證

```bash
client:~ # zypper in tftp
client:~ # tftp 192.168.1.10
tftp> get hostname
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 11-3-1 | Server 放入測試檔，Client `get` 成功，用 `cat hostname` 核對內容     |
| 11-3-2 | 先單點 `tftp get` 成功，再進入 PXE 整機測試（見練習 13）             |

### 作答／驗收（練習 11）

1. `tftp` 服務 active、防火牆已放行。
2. Client 取得的檔案內容與 Server 一致。

---

## 練習 12. MUNGE：叢集信任 Client／Server 架設

MUNGE 用**同一把共享金鑰**證明訊息來自可信主機且未被竄改。所有節點時間需同步（接回練習 3）。

### 12-1. 概念

| 小題  | 問題                                                                 | 作答 |
| ----- | -------------------------------------------------------------------- | ---- |
| 12-1-1 | 說明為何時間不同步會導致 MUNGE 驗證失敗                              |      |
| 12-1-2 | 為什麼金鑰外洩等於認證被攻破？傳輸應走什麼通道、檔案權限應如何？     |      |
| 12-1-3 | `munge -n \| unmunge` 驗證的是哪兩件事？                             |      |

### 12-2. Server 架設

```bash
server:~ # zypper in munge
server:~ # mungekey -c -k /etc/munge/munge.key
server:~ # chown munge:munge /etc/munge/munge.key
server:~ # chmod 400 /etc/munge/munge.key
server:~ # systemctl enable --now munge.service
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 12-2-1 | 寫出建立金鑰、設權限、啟動服務的完整順序                             |
| 12-2-2 | `munge.service` active                                               |

### 12-3. Client 架設與跨機驗證

```bash
client:~ # zypper in munge
server:~ # scp /etc/munge/munge.key root@192.168.1.20:/etc/munge/
client:~ # chown munge:munge /etc/munge/munge.key
client:~ # chmod 400 /etc/munge/munge.key
client:~ # systemctl enable --now munge.service

server:~ # munge -n | ssh 192.168.1.20 unmunge
```

| 小題  | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 12-3-1 | 以 `scp`（或其他安全通道）複製**同一把** key，權限正確後啟動         |
| 12-3-2 | 從 Server `munge` 到 Client `unmunge` 成功                           |
| 12-3-3 | （思考）若 Chrony 未同步，這步最可能出現什麼現象？                   |

### 作答／驗收（練習 12）

1. 兩機 `munge.service` active，金鑰權限正確。
2. 跨機 `munge | unmunge` 成功輸出。
3. 能連結「時間同步」與驗證失敗。

---

## 練習 13. PXE：DHCP + TFTP + HTTP 自動安裝鏈

PXE 讓主機開機時經網路取得 bootloader 與安裝核心。典型組合：

**DHCP（`next-server`／`filename`）+ TFTP（bootloader／kernel／initrd）+ HTTP／NFS（安裝媒體）**

本練習以 **Server 當 PXE 伺服器**、**Client 改網路開機**（或第三台測試機）。無 ISO／無法改開機順序時，至少完成檔案準備、DHCP 片段與 `tftp get` 單點驗證，並畫出流程。

### 13-1. 流程與概念

| 小題   | 問題                                                                 | 作答 |
| ------ | -------------------------------------------------------------------- | ---- |
| 13-1-1 | 畫出（或逐步文字描述）DHCP → TFTP → HTTP 的開機流程                  |      |
| 13-1-2 | 說明為何要用 `option arch` 分開 UEFI／BIOS 的 `filename`             |      |
| 13-1-3 | `next-server` 與 `filename` 各代表什麼？                             |      |
| 13-1-4 | `pxelinux.cfg/default` 裡 `KERNEL`、`initrd`、`install=http://...` 意義 |    |
| 13-1-5 | 若 Client 停在 PXE 且 TFTP 逾時，應先檢查哪些項目？                  |      |

### 13-2. Server：準備 TFTP 開機檔

路徑與 ISO 檔名依實際媒體調整（範例為 SLE-15）：

```bash
server:~ # mkdir -p /srv/tftpboot/sle15
server:~ # mount -oloop SLE-15-*-x86_64-*.iso /mnt
server:~ # cp /mnt/boot/x86_64/loader/linux /srv/tftpboot/sle15/
server:~ # cp /mnt/boot/x86_64/loader/initrd /srv/tftpboot/sle15/
server:~ # cp /usr/share/syslinux/pxelinux.0 /srv/tftpboot/
server:~ # mkdir -p /srv/tftpboot/pxelinux.cfg
server:~ # vi /srv/tftpboot/pxelinux.cfg/default
server:~ # chown -R tftp:tftp /srv/tftpboot
server:~ # chmod -R 755 /srv/tftpboot
server:~ # tftp 127.0.0.1 -c get pxelinux.0
```

| 小題   | 任務                                                                 |
| ------ | -------------------------------------------------------------------- |
| 13-2-1 | 準備 kernel／initrd／`pxelinux.0`（或 UEFI 對應檔）                  |
| 13-2-2 | 撰寫 `pxelinux.cfg/default`，`install=` 指向下一步 HTTP              |
| 13-2-3 | 本機 `tftp get pxelinux.0` 成功                                      |

### 13-3. Server：DHCP 整合 PXE

在練習 7 的 `dhcpd.conf` 中加入（概念，請合併進你的 subnet，不要複製出第二個衝突 subnet）：

```conf
option arch code 93 = unsigned integer 16;
subnet 192.168.1.0 netmask 255.255.255.0 {
  next-server 192.168.1.10;
  if option arch = 00:09 {
    filename "/EFI/BOOT/bootx64.efi";
  } else {
    filename "pxelinux.0";
  }
}
```

| 小題   | 任務                                                                 |
| ------ | -------------------------------------------------------------------- |
| 13-3-1 | 寫出你實際採用的 `next-server` 與 BIOS／UEFI `filename`              |
| 13-3-2 | `dhcpd -t` 後 reload `dhcpd`                                         |

### 13-4. Server：HTTP 提供安裝媒體 + Client 網路開機

```apache
# /etc/apache2/conf.d/sle15.conf 概念
Alias /sle15 "/mnt"
<Directory "/mnt">
    Options Indexes FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>
```

```bash
server:~ # apachectl configtest
server:~ # systemctl reload apache2
client:~ # curl -I http://192.168.1.10/sle15/
```

| 小題   | 任務                                                                 |
| ----- | -------------------------------------------------------------------- |
| 13-4-1 | HTTP 能列出或取到安裝媒體路徑；Client `curl -I` 非 4xx／5xx          |
| 13-4-2 | 放行 DHCP／TFTP／HTTP（前面練習應已開；此處再核對一次）              |
| 13-4-3 | （實作加分）Client 開機選單改 PXE，看到安裝選單或 kernel 開始載入    |
| 13-4-4 | 權限與路徑必須相對 TFTP root；說明「DHCP 有位址但 TFTP 路徑不對」的排查 |

### 作答／驗收（練習 13）

1. 必做：流程說明、`option arch` 理由、TFTP 單點 `get`、DHCP 關鍵參數、HTTP `curl`。
2. 加分：Client 實際網路開機看到選單。
3. TFTP 逾時時能列出檢查項（服務、防火牆、filename、權限、BIOS／UEFI 分派）。

---

## 練習 14. 綜合：服務閉環與排查

不必把所有服務同時開在正式環境；用表格證明你知道「Server 做什麼、Client 測什麼」。

### 14-1. 每個服務的 Client 驗證指令

| 服務    | Server 關鍵服務名      | 防火牆（service／port） | Client 驗證指令（請填） |
| ------- | ---------------------- | ----------------------- | ----------------------- |
| SSH     | `sshd`                 |                         |                         |
| Chrony  | `chronyd`              |                         |                         |
| NIS     | `ypserv`／`ypbind`     |                         |                         |
| NFS     | `nfs-server`           |                         |                         |
| DNS     | `named`                |                         |                         |
| DHCP    | `dhcpd`                |                         |                         |
| HTTP    | `apache2`              |                         |                         |
| FTP     | `vsftpd`               |                         |                         |
| TFTP    | `tftp`                 |                         |                         |
| MUNGE   | `munge`                | （多為主機間金鑰，非埠） |                       |
| PXE     | dhcpd+tftp+apache2     |                         |                         |

### 14-2. 情境題（寫檢查順序即可）

| 情境 | 你的排查順序（至少三步） |
| ---- | ------------------------ |
| Client SSH 連不上，但 Server `sshd` 是 active | |
| NFS 本機能 `exportfs`，Client `mount` 逾時 | |
| `named` 有起來，Client `dig @server` 無 ANSWER | |
| DHCPD active，Client 一直自行連線（APIPA）或拿不到 IP | |
| FTP 能登入，`get`／`put` 卡住 | |
| PXE 已拿到 IP，停在 TFTP timeout | |
| MUNGE `unmunge` 失敗，兩機都有 key | |

### 14-3. 建議實作順序自評

本章建議順序：systemd → SSH → Chrony → DNS／DHCP → NFS／FTP／TFTP → Apache → NIS → MUNGE → PXE。

寫下你實際完成了哪些 **Client／Server 架設**（打勾）：

- [ ] 練習 2 SSH
- [ ] 練習 3 Chrony
- [ ] 練習 4 NIS
- [ ] 練習 5 NFS
- [ ] 練習 6 DNS
- [ ] 練習 7 DHCP
- [ ] 練習 8 Apache
- [ ] 練習 9 FTP
- [ ] 練習 11 TFTP
- [ ] 練習 12 MUNGE
- [ ] 練習 13 PXE（含流程即可，或加分實機開機）

### 作答／驗收（練習 14）

1. 14-1 表格填完（指令可簡短）。
2. 14-2 至少完成四個情境。
3. 強調：服務「能啟動」不算完成，必須 **Client 驗證閉環**；改設定先備份、能 `configtest`／`sshd -t`／`named-checkzone`／`dhcpd -t` 的都要先檢查再 reload。
