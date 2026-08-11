# ch7. SUSE Linux Enterprise Server 15 - Script

系統管理與科學計算環境常需把重複操作寫成腳本。Linux 上常見 shell 包括 **Bash**（預設最廣）、**tcsh**（部分傳統 HPC／舊軟體環境）、**Zsh**（互動體驗強）。本章說明三種 shell 的啟動檔載入邏輯、變數、流程控制、函數（或替代作法）、管線與字串處理，並附練習與總結。

撰寫腳本前先確認 shebang（如 `#!/bin/bash`）與執行權限，避免「在 A shell 測試通過、用 B shell 執行卻失敗」。

---

## 7-1. Bash

Bash（Bourne Again SHell）是多數 Linux 發行版的預設登入／腳本 shell，語法兼容 Bourne shell 並擴充陣列、管線與參數展開等能力。

### 7-1-1. 啟動檔載入邏輯

執行 bash 時，會依 **login / non-login** 與 **interactive / non-interactive** 決定讀哪些檔案。

#### 登入 shell（login shell）

典型情境：`ssh`、文字主控台登入、`bash -l`。

讀取順序概念：

1. `/etc/profile`（系統全域）
2. 使用者檔（擇一，優先順序）：  
   `~/.bash_profile` → `~/.bash_login` → `~/.profile`

#### 非登入 shell（non-login shell）

典型情境：已登入後再開終端分頁、多數 GUI terminal 預設。

1. `/etc/bash.bashrc`（系統；檔名依發行版可能為 `/etc/bashrc`）
2. `~/.bashrc`（使用者）

#### 互動 + 登入（常見：SSH）

登入流程主要走 profile 系檔案。為了讓 alias／function 在登入後也生效，`~/.profile` 或 `~/.bash_profile` 內通常會手動載入 `~/.bashrc`：

```bash
if [ -f ~/.bashrc ]; then
  . ~/.bashrc
fi
```

#### 流程示意

```text
                 ┌──────────────┐
                 │  啟動 Bash   │
                 └──────┬───────┘
                        │
          ┌─────────────┴─────────────┐
          │                           │
    [登入 shell]                 [非登入 shell]
 (ssh, tty, bash -l)         (開新終端、一般 bash)

          │                           │
   ┌──────┴───────┐             ┌─────┴─────┐
   │              │             │           │
 [互動式]     [非互動式]     [互動式]    [非互動式]

   │              │             │           │
 讀 profile 系     腳本常略過      讀 bashrc 系    腳本常略過
 /etc/profile      互動專用設定    /etc/bash.bashrc  互動專用設定
 ~/.bash_profile                  ~/.bashrc
 或 ~/.bash_login
 或 ~/.profile
 （常再 source ~/.bashrc）
```

| 類型            | 意義                                      |
| --------------- | ----------------------------------------- |
| login           | 建立完整登入環境（PATH、umask、系統政策） |
| interactive     | 需要提示字元、補完、alias 等互動體驗      |
| non-interactive | 腳本執行；應自給自足，少依賴互動設定      |

- **意義**: 「SSH 有 alias、腳本卻沒有」通常是非互動腳本未讀 `bashrc`，或 profile 未 source `bashrc`。

### 7-1-2. 執行方式

```bash
#!/bin/bash
echo "Hello, world!"
```

```bash
# 方式 1：明確指定解譯器
linux:~ $ bash hello.sh

# 方式 2：給執行權限，依 shebang 執行
linux:~ $ chmod +x hello.sh
linux:~ $ ./hello.sh
```

| 步驟                      | 意義                                                   |
| ------------------------- | ------------------------------------------------------ |
| `#!/bin/bash`             | 告訴核心用哪個解譯器                                   |
| `bash hello.sh`           | 不需 +x，適合測試                                      |
| `chmod +x` + `./hello.sh` | 可當指令呼叫；需注意 `PATH` 是否包含 `.`（通常不包含） |

### 7-1-3. 變數

```bash
name="Alice"
echo "Hello $name"

export myvar="test"
echo $myvar

set       # 列出 shell 變數（含函式等，輸出量大）
env       # 列出環境變數

fruits=("apple" "banana" "cherry")
echo ${fruits[0]}      # apple（索引從 0）
echo ${fruits[@]}      # 全部元素
echo ${#fruits[@]}     # 長度

# 關聯陣列（Bash 4+）
declare -A capitals
capitals[TW]="Taipei"
capitals[JP]="Tokyo"
echo ${capitals[TW]}
for k in "${!capitals[@]}"; do
  echo "$k -> ${capitals[$k]}"
done
```

| 重點         | 意義                     |
| ------------ | ------------------------ |
| `name=value` | `=` 兩側**不能有空白**   |
| `export`     | 變為環境變數，子行程可見 |
| 索引陣列     | 從 `0` 起算              |
| `declare -A` | 關聯陣列（key → value）  |

### 7-1-4. 條件判斷與迴圈

```bash
x=5
if [ $x -gt 3 ]; then
  echo "x 大於 3"
else
  echo "x 小於等於 3"
fi

read -p "輸入 y/n: " ans
case $ans in
  y|Y) echo "Yes" ;;
  n|N) echo "No" ;;
  *) echo "其他輸入" ;;
esac

for i in 1 2 3; do
  echo "數字: $i"
done

count=1
while [ $count -le 3 ]; do
  echo "次數: $count"
  ((count++))
done
```

| 類型 | 運算子                              |
| ---- | ----------------------------------- |
| 整數 | `-eq` `-ne` `-gt` `-lt` `-ge` `-le` |
| 字串 | `=` `!=` `-z`（空）`-n`（非空）     |
| 檔案 | `-f` 一般檔、`-d` 目錄、`-e` 存在   |

- `[ ]` 是 `test` 的語法糖；進階建議逐步熟悉 `[[ ]]`（較少分詞陷阱）。
- 變數應加引號，如 `[ -n "$ans" ]`，避免空值造成語法錯誤。

### 7-1-5. 函數

```bash
hello() {
  echo "Hello $1"
}
hello Alice

add() {
  return $(($1 + $2))
}
add 3 4
echo $?    # 注意：$? 僅 0–255
```

| 概念             | 意義                                          |
| ---------------- | --------------------------------------------- |
| `$1` `$2`        | 位置參數                                      |
| `return`         | 回傳狀態碼，不是任意整數結果通道              |
| `$?`             | 上一個命令結束狀態；大於 255 會溢位截斷       |
| 需要回傳數值字串 | 用 `echo` 輸出，再以 `result=$(add 3 4)` 承接 |

### 7-1-6. Pipeline（管線）

#### 基本單向管線

```text
cmd1 | cmd2 | cmd3
[stdin] → cmd1 ──stdout──► cmd2 ──stdout──► cmd3 → [stdout]
```

```bash
cat /etc/passwd | grep bash | wc -l
```

- 左側 **stdout** 接到右側 **stdin**。
- 預設 **stderr** 不進管線，仍輸出到終端。

#### stderr 與合併

```bash
ls /not_exist | wc -l          # 錯誤訊息上螢幕；wc 可能數到 0 行
ls /etc /not_exist 2>&1 | grep passwd
cmd1 |& cmd2                   # Bash：等同 2>&1 |
```

| 寫法       | 意義                         |
| ---------- | ---------------------------- | ------------------------ |
| `2>&1`     | 把 stderr 指向 stdout 的目標 |
| `          | &`                           | stdout+stderr 一起進管線 |
| `&> file`  | 兩者覆寫導向檔案             |
| `&>> file` | 兩者追加導向檔案             |

#### `tee` 分流

```bash
echo "Hello World" | tee output.txt | tr 'a-z' 'A-Z'
```

```text
cmd1 ──stdout──► tee file ─► cmd2
                   │
                   └─► 寫入檔案
```

多重行程替換：

```bash
cmd1 | tee >(cmd2) >(cmd3)
```

- **意義**: 一邊保存日誌、一邊繼續處理；除錯與審計常用。

### 7-1-7. 參數／字串展開

```bash
${var:-default}   # 未定義或空 → 使用 default（不寫回 var）
${var:=default}   # 未定義或空 → 設成 default
${var:?msg}       # 未定義或空 → 報錯結束
${var:+alt}       # 已定義且非空 → 使用 alt

file="backup_2025.tar.gz"
${file%.gz}        # backup_2025.tar
${file%%.tar.gz}   # backup_2025
${file#backup_}    # 2025.tar.gz
${file##*_}        # gz

str="abcdef"
${#str}            # 6
${str:2:3}         # cde（Bash 從 0 起算）

text="apple banana apple"
${text/apple/pear}    # 只替第一個
${text//apple/pear}   # 全部替換
```

| 形式             | 意義                   |
| ---------------- | ---------------------- |
| `%`／`%%`        | 去後綴（最短／最長）   |
| `#`／`##`        | 去前綴（最短／最長）   |
| `${#str}`        | 長度                   |
| `${str:pos:len}` | 子字串（Bash 0-based） |

### 學習心得：Bash

- 先分清 login／interactive，才能解釋「為何設定檔沒生效」。
- 腳本應自帶 shebang、少依賴 alias，需要的函式請寫進腳本或明確 `source`。
- 管線預設不管 stderr；日誌收集時常要 `2>&1` 或 `|&`。
- `return`／`$?` 不是通用整數回傳通道，數值結果應用 stdout。

### 練習：Bash

1. 說明 SSH 登入時，為何常在 `~/.bash_profile` 裡 `source ~/.bashrc`。
2. 寫一支腳本：讀取檔名參數，若不存在則用 `${1:?need file}` 報錯。
3. 用管線統計 `/etc/passwd` 中 shell 為 `bash` 的列數，並把完整輸出（含可能錯誤）存到日誌。
4. 比較 `${file%.*}` 與 `${file%%.*}` 對 `a.b.c` 的結果。

---

## 7-2. tcsh

tcsh 是 csh 的強化版，在部分學術／HPC 與舊軟體（如某些 Gaussian 腳本）環境仍常見。語法與 Bash 差異大，**不能假設 Bash 腳本可直接當 tcsh 跑**。

### 7-2-1. 啟動邏輯

#### Login shell

情境：`ssh`、`su -`、tty 登入。

讀取順序：

1. `/etc/csh.cshrc`
2. `/etc/csh.login`
3. `~/.cshrc`（或若存在則優先 `~/.tcshrc`）
4. `~/.login`
5. （部分系統）`~/.login_conf`

- 若存在 **`~/.tcshrc`**，通常**不會再讀** `~/.cshrc`。
- `.cshrc`／`.tcshrc` 適合 alias、prompt；`.login` 適合 PATH、一次性初始化。

#### Non-login shell

1. `/etc/csh.cshrc`
2. `~/.cshrc` 或 `~/.tcshrc`

#### Logout

login shell 結束時可讀 `~/.logout` 做清理。

### 7-2-2. 執行方式

```tcsh
#!/bin/tcsh
echo "Hello from tcsh"
```

```tcsh
linux:~ $ tcsh myscript.csh
linux:~ $ chmod +x myscript.csh
linux:~ $ ./myscript.csh
```

### 7-2-3. 變數

```tcsh
set name = "Alice"
echo "Hello $name"

setenv PATH "$PATH:/usr/local/bin"
echo $PATH

set fruits = (apple banana cherry)
echo $fruits[1]     # apple（索引從 1）
echo $fruits
echo $#fruits
```

| 與 Bash 差異    | 說明                   |
| --------------- | ---------------------- |
| `set`／`setenv` | shell 變數 vs 環境變數 |
| 陣列索引        | **從 1 開始**          |
| 關聯陣列        | **不支援**             |
| `set a = b`     | `=` 兩側空白是常見寫法 |

### 7-2-4. 判斷與迴圈

```tcsh
set x = 5
if ( $x > 3 ) then
    echo "x 大於 3"
else
    echo "x 小於等於 3"
endif

switch ( $ans )
case y:
case Y:
    echo "Yes"
    breaksw
case n:
case N:
    echo "No"
    breaksw
default:
    echo "其他輸入"
    breaksw
endsw

foreach i ( 1 2 3 4 5 )
    echo "數字: $i"
end

set count = 1
while ( $count <= 3 )
    echo "次數: $count"
    @ count++
end
```

| 重點                         | 意義                       |
| ---------------------------- | -------------------------- |
| `if ( ) then` … `endif`      | 條件必須用括號             |
| `switch`／`breaksw`／`endsw` | 對應 Bash `case`           |
| `foreach`                    | 對清單迭代                 |
| `@ count++`                  | 算術；不是 Bash 的 `(( ))` |

### 7-2-5. 「函數」的替代作法

tcsh **沒有**真正等同 Bash 的函式機制，實務常用：

#### 1. `alias` 模擬

```tcsh
alias hello 'echo "Hello \!:1"'
hello Alice
```

（實際參數引用方式依 tcsh alias 歷史參數語法；複雜邏輯不建議硬塞 alias。）

#### 2. 獨立腳本

```tcsh
#!/bin/tcsh
@ sum = $1 + $2
echo $sum
```

```tcsh
linux:~ $ ./add.csh 3 4
```

- **意義**: 可重用邏輯模組化成檔案，比 alias 可維護。

### 7-2-6. Pipeline 行為差異

```tcsh
echo "abc" | read line
echo $line
```

| Shell | 常見結果                                         |
| ----- | ------------------------------------------------ |
| Bash  | `$line` 常為空（管線右側多在子 shell）           |
| tcsh  | 最後的 `read` 可在當前 shell，`$line` 可為 `abc` |

- **意義**: 同樣「管線 + read」在不同 shell 語意不同，移植腳本必須重測。

### 7-2-7. 前綴／後綴修飾子（modifiers）

語法：`$var:modifier`

```tcsh
set file = /usr/local/bin/test.sh
echo $file:h    # /usr/local/bin
echo $file:t    # test.sh
echo $file:r    # /usr/local/bin/test
echo $file:e    # sh

set name = "hello_world.txt"
echo $name:r
echo $name:e
echo $name:gs/_/-/
echo $name:u
```

| 修飾子         | 意義                   |
| -------------- | ---------------------- |
| `:h`           | head，去掉最後一層路徑 |
| `:t`           | tail，取最後元件       |
| `:r`           | 去掉副檔名             |
| `:e`           | 取副檔名               |
| `:gs/old/new/` | 全域取代               |
| `:u`／`:l`     | 大寫／小寫             |
| `:q`           | 加引號，避免空白分詞   |
| `:x`           | 展開成多參數           |

### 學習心得：tcsh

- 與 Bash 的差異是「語法家族」等級，不是小方言。
- 陣列從 1 起算、無關聯陣列、無真函式，設計腳本時要換思維。
- 維護舊科學軟體環境時，搞清楚啟動檔（`.tcshrc` vs `.cshrc`）可少打許多槍。

### 練習：tcsh

1. 列出 login tcsh 的設定檔讀取順序，並說明 `.tcshrc` 與 `.cshrc` 關係。
2. 用 `foreach` 印出三個檔名的 `:t` 與 `:e`。
3. 解釋為何 Bash 的 `f(){ ...; }` 不能直接貼進 tcsh。

---

## 7-3. Zsh

Zsh（Z Shell）結合 Bash 易用性與更強互動／展開能力，常搭配 Oh My Zsh 等框架；作為登入 shell 或腳本解譯器皆可。

### 7-3-1. 啟動檔

| 檔案       | 何時執行                  | 用途                                          |
| ---------- | ------------------------- | --------------------------------------------- |
| `zshenv`   | 每次啟動最先              | 環境變數（如 `PATH`）；不宜放 alias／提示字元 |
| `zprofile` | login shell               | 類似 Bash profile 的登入環境                  |
| `zshrc`    | interactive shell         | alias、prompt、補完、函式                     |
| `zlogin`   | login，在 `zprofile` 之後 | 登入後動作、歡迎訊息                          |
| `zlogout`  | login 結束                | 登出清理                                      |

系統與使用者檔通常成對出現於 `/etc/` 與 `$HOME`（如 `~/.zshrc`）。

| 建議               | 意義                                 |
| ------------------ | ------------------------------------ |
| 精簡 `zshenv`      | 每個腳本都會讀，過重會拖慢非互動執行 |
| 互動設定放 `zshrc` | 避免污染腳本環境                     |

### 7-3-2. 執行方式

```zsh
#!/bin/zsh
echo "Hello from zsh"
```

```zsh
linux:~ $ zsh hello.zsh
linux:~ $ chmod +x hello.zsh
linux:~ $ ./hello.zsh
```

### 7-3-3. 變數與陣列（要點）

- 指派類似 Bash：`name="Alice"`。
- 陣列預設索引常為 **1-based**（與 Bash 0-based 不同）。
- 關聯陣列：`typeset -A`／`declare -A`（視設定）。
- 條件判斷推薦 **`[[ ]]`**，支援較豐富的字串與模式比對，分詞陷阱較少。

```zsh
x=5
if [[ $x -gt 3 ]]; then
  echo "x 大於 3"
fi
```

### 7-3-4. 函數

```zsh
hello() {
  echo "Hello, $1"
}
hello Linux

add() {
  echo $(($1 + $2))
}
result=$(add 5 3)
echo $result   # 8
```

- 以 stdout 回傳可計算結果，比單純依賴 `return`／`$?` 實用。

### 7-3-5. Pipeline

基本概念與 Bash 相同：stdout → 下一命令 stdin。

```zsh
cat /etc/passwd | grep "bash" | wc -l
ls /notfound | wc -l          # stderr 預設不進管線
make |& tee build.log         # stdout+stderr
ls | tee list.txt | grep ".txt"
echo "Hello" | tee >(tr 'a-z' 'A-Z') >(rev) >(wc -c)
diff <(ls dir1) <(ls dir2)    # process substitution
```

| 技巧           | 意義                           |
| -------------- | ------------------------------ | -------------------- |
| `              | &`                             | 合併錯誤與輸出進管線 |
| `tee`          | 複製輸出到檔案與下游           |
| `<( )`／`>( )` | 行程替換，讓命令當「檔案」參數 |

### 7-3-6. 前綴／後綴與字串展開（精選）

Zsh 提供豐富的展開；下列與實務最相關（細節以 `man zshexpn` 為準）：

| 語法                                                       | 說明                       | 範例概念                         |
| ---------------------------------------------------------- | -------------------------- | -------------------------------- |
| `${#var}`                                                  | 字串長度／陣列元素數       | `file="abc.txt"` → 7             |
| `${var%pat}`／`${var%%pat}`                                | 去最短／最長後綴           | `report.txt` → `report`          |
| `${var#pat}`／`${var##pat}`                                | 去最短／最長前綴           | `*.txt` 類切割副檔名             |
| `${var:pos}`／`${var:pos:len}`                             | 子字串（Zsh 常為 1-based） | 與 Bash 0-based 不同，移植需小心 |
| `${arr[i]}`／`${arr[-1]}`                                  | 陣列元素／倒數             | `${arr[2,4]}` 切片               |
| `${var/pat/repl}`／`${var//pat/repl}`                      | 替一處／全部               | 底線與空白整理                   |
| `${var/#pat/repl}`／`${var/%pat/repl}`                     | 只替前綴／後綴             | 正規化檔名                       |
| `${var:u}`／`${var:l}`                                     | 大寫／小寫                 | 大小寫正規化                     |
| `${var:-def}`／`${var:=def}`／`${var:+alt}`／`${var:?msg}` | 預設值與嚴格檢查           | 參數防呆                         |
| `${var:q}`                                                 | 引號化                     | 避免空白分詞                     |

### 學習心得：Zsh

- 啟動檔分層清楚：環境放 `zshenv`／`zprofile`，互動放 `zshrc`。
- 索引與子字串基準可能與 Bash 不同，**跨 shell 共用函式庫風險高**。
- `[[ ]]`、行程替換與強大展開，適合互動與進階腳本，但仍應保持可讀性。

### 練習：Zsh

1. 說明 `zshenv` 與 `zshrc` 不該混放哪些設定。
2. 用 `tee` 與 `|&` 把建置輸出完整存成 `build.log`。
3. 比較同一句 `${name:2:3}` 在 Bash 與 Zsh 可能差異，並查 man 驗證。

---

## 7-4. 三殼對照（速查）

| 項目      | Bash           | tcsh                 | Zsh              |
| --------- | -------------- | -------------------- | ---------------- |
| 典型用途  | 系統腳本預設   | 舊環境／特定軟體     | 互動＋進階腳本   |
| 陣列索引  | 0-based        | 1-based              | 常為 1-based     |
| 關聯陣列  | `declare -A`   | 無                   | 支援             |
| 真函式    | 有             | 無（alias／外檔）    | 有               |
| 條件括號  | `[ ]`／`[[ ]]` | `( )` + `then/endif` | 建議 `[[ ]]`     |
| 算術遞增  | `((count++))`  | `@ count++`          | 類似 Bash        |
| 字串修飾  | `${var%}` 系   | `$var:h` 系          | 兩者風格皆豐富   |
| 管線+read | 多在子 shell   | 常留在當前 shell     | 行為需測（可設） |

---

## 步驟總結

學習與撰寫 shell 腳本建議順序：

1. **確認目標解譯器**（shebang）與版本（Bash 4+ 才有關聯陣列等）
2. **搞懂啟動檔**：何時讀 profile／bashrc／cshrc／zshrc
3. **先寫最小可執行檔**：`echo` → 變數 → `if`／迴圈
4. **再學管線與重導向**：stdout／stderr／`tee`
5. **最後精熟字串展開**：去前後綴、預設值、取代
6. **跨 shell 需求**先做對照表，不要直接複製貼上
7. **正式環境**把常用函式與環境改以明確 `source` 或 module／文件管理

---

## 總結

本章建立腳本能力的核心地圖：Bash 是系統自動化的主幹，tcsh 仍出現在特定遺產環境，Zsh 則強化互動與展開力。真正造成故障的，往往不是「不會寫 for 迴圈」，而是**啟動檔沒載入、子 shell 讓變數消失、stderr 沒進日誌、索引基準與修飾語法套錯家族**。掌握登入邏輯、變數與流程、管線與字串處理後，即可把第 2～6 章的操作組合成可重現的自動化流程，並在 SLES 管理與科學計算環境中穩定使用。
