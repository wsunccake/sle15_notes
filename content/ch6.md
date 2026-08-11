# ch6. SUSE Linux Enterprise Server 15 - Science Computing

科學計算（Science Computing）環境通常需要：**高效能編譯器 / MPI 工具鏈**，以及依授權安裝的領域應用軟體（量子化學、固態物理、分子動力學等）。本章說明 Intel oneAPI 與 NVIDIA HPC SDK 的安裝與使用步驟，並介紹 VASP、Gaussian、GROMACS、Quantum ESPRESSO 的背景與建置要點。實作前建議先具備第 5 章的 Lmod / Slurm 觀念，以便管理多版本工具鏈與批次作業。

參考總表：[List of quantum chemistry and solid-state physics software](https://en.wikipedia.org/wiki/List_of_quantum_chemistry_and_solid-state_physics_software)

---

## 6-1. Compiler（編譯器工具鏈）

科學應用常見語言為 **C / C++ / Fortran**，平行化則多依賴 **MPI**（Message Passing Interface），GPU 加速則涉及 CUDA / OpenACC / OpenMP offload 等。選擇編譯器時，需同時考慮：授權、目標硬體（CPU / GPU）、與應用軟體的官方支援組合。

### 6-1-1. Intel Compiler / oneAPI

#### 歷史演進（精要）

| 階段                     | 說明                                                                                            |
| ------------------------ | ----------------------------------------------------------------------------------------------- |
| 傳統 Intel Compiler      | 長期以 `icc` / `icpc` / `ifort` 服務高效能 CPU 最佳化與 Fortran 科學社群                        |
| Intel Parallel Studio XE | 整合編譯器、MKL、MPI、分析工具的商業套件時代                                                    |
| **Intel oneAPI**         | 轉向異質計算統一開發模型；HPC 相關元件收斂於 **oneAPI HPC Toolkit**                             |
| Classic → LLVM 新前端    | 新一代指令逐漸以 `icx` / `icpx` / `ifx` 為主（LLVM 基礎），舊名 `icc` / `ifort` 進入過渡 / 退場 |

- **意義**: 現今文件若仍寫 `icc` / `ifort`，在新 toolkit 上常需改成 `icx` / `ifx`，makefile 也要同步調整（見後文 VASP）。

#### 下載與安裝

下載頁：[Intel® oneAPI HPC Toolkit](https://www.intel.com/content/www/us/en/developer/tools/oneapi/hpc-toolkit-download.html)

```bash
sle15:~ # sh intel-oneapi-hpc-toolkit-<yyyy>.<m>.<xxx>_offline.sh
```

| 步驟                        | 意義                                           |
| --------------------------- | ---------------------------------------------- |
| 執行 offline installer      | 在無外網或需重現安裝的環境部署完整元件         |
| 版本字串 `<yyyy>.<m>.<xxx>` | 對應發行年月與建置號，文件與 module 路徑需一致 |

#### 載入環境

```bash
sle15:~ $ ls /opt/intel/oneapi
sle15:~ $ source /opt/intel/oneapi/<yyyy>.<m>/oneapi-vars.sh
sle15:~ $ echo $ONEAPI_ROOT
```

| 動作                        | 意義                                  |
| --------------------------- | ------------------------------------- |
| `source .../oneapi-vars.sh` | 設定 `PATH`、函式庫、`ONEAPI_ROOT` 等 |
| `echo $ONEAPI_ROOT`         | 確認環境已生效，供後續編譯與腳本引用  |

- 長期使用建議改以 **Lmod modulefile** 包裝，避免每個 session 手動 `source`。

#### C / C++ / Fortran 編譯

```bash
# C
sle15:~ $ which icx
sle15:~ $ icx -o hello hello.c

# C++
sle15:~ $ which icpx
sle15:~ $ icpx -o hello hello.cpp

# Fortran
sle15:~ $ which ifx
sle15:~ $ ifx -o hello -fixed hello.f     # Fortran 77 固定格式
sle15:~ $ ifx -o hello hello.f90          # Fortran 90 自由格式
```

| 編譯器   | 用途                        |
| -------- | --------------------------- |
| `icx`    | C（oneAPI 新前端）          |
| `icpx`   | C++                         |
| `ifx`    | Fortran                     |
| `-fixed` | 告知固定格式 Fortran 原始碼 |

#### MPI 包裝器

通用 MPI wrapper（可指定底層編譯器）：

```bash
sle15:~ $ mpicc  -o mpi_pi [-cc=<c_compiler>]       mpi_pi.c
sle15:~ $ mpicxx -o mpi_pi [-cxx=<c++_compiler>]    mpi_pi.cpp
sle15:~ $ mpif77 -o mpi_pi [-fc=<fortran_compiler>] mpi_pi.f
sle15:~ $ mpif90 -o mpi_pi [-fc=<fortran_compiler>] mpi_pi.f90
```

Intel 專用 wrapper（直接對應 Intel 工具鏈）：

```bash
sle15:~ $ mpiicx  -o mpi_pi mpi_pi.c
sle15:~ $ mpiicpx -o mpi_pi mpi_pi.cpp
sle15:~ $ mpiifx  -o mpi_pi mpi_pi.f
sle15:~ $ mpiifx  -o mpi_pi mpi_pi.f90

sle15:~ $ mpirun -n <NPROC> ./mpi_pi
```

| 項目                            | 意義                                                                 |
| ------------------------------- | -------------------------------------------------------------------- |
| `mpiicx` / `mpiicpx` / `mpiifx` | 以 Intel 編譯器為底的 MPI 編譯入口                                   |
| `mpirun -n <NPROC>`             | 啟動 `<NPROC>` 個 MPI process                                        |
| 與 Slurm 整合                   | 正式叢集常用 `srun` / Slurm PMI 啟動，而非互動 `mpirun` 佔滿登入節點 |

#### Intel 工具鏈建議步驟

1. 安裝 HPC Toolkit
2. `source oneapi-vars.sh` 或 `ml add` 對應模組
3. `which icx icpx ifx mpirun` 確認路徑
4. 先編譯序列 `hello`，再編譯 MPI 範例
5. 小規模 `mpirun -n 2` 驗證後，再接到應用軟體建置

### 學習心得：Intel oneAPI

- 科學軟體建置失敗，常見原因是「文件寫舊編譯器名稱、環境卻是新 oneAPI」。
- MPI wrapper 與底層編譯器必須同一套，混用 GCC / Intel 標頭與函式庫容易連結失敗。
- `ONEAPI_ROOT` 與 module 化是多人叢集可重現環境的關鍵。

### 練習：Intel

1. 對照說明 `icc` / `ifort` 與 `icx` / `ifx` 的世代差異。
2. 寫出編譯 C MPI 程式的兩種方式：`mpicc -cc=icx` 與 `mpiicx`。
3. 解釋為何要先 `echo $ONEAPI_ROOT` 再開始編譯應用。

---

### 6-1-2. NVIDIA HPC SDK

#### 歷史演進（精要）

| 階段                  | 說明                                                                                                    |
| --------------------- | ------------------------------------------------------------------------------------------------------- |
| Portland Group（PGI） | 以 `pgcc` / `pgc++` / `pgfortran` 聞名，強化 GPU / 加速器編譯                                           |
| NVIDIA 收購 PGI       | 編譯器技術併入 NVIDIA 工具生態                                                                          |
| CUDA Toolkit 並行發展 | GPU runtime、函式庫、驅動生態的基礎                                                                     |
| **NVIDIA HPC SDK**    | 整合 NVIDIA 編譯器（`nvc` / `nvc++` / `nvfortran`）、CUDA、數學函式庫、MPI 等，面向 HPC / AI / 加速運算 |

- **意義**: 目標若含 GPU（或 OpenACC），NVIDIA HPC SDK 常是首選工具鏈；純 CPU 且依賴 Intel MKL / 特定 Fortran 行為時，則可能仍選 oneAPI。

#### 下載與安裝

下載頁：[NVIDIA HPC SDK](https://developer.nvidia.com/hpc-sdk)

```bash
sle15:~ # tar zxf nvhpc_2025_257_Linux_x86_64_cuda_12.9.tar.gz
sle15:~ # cd nvhpc_2025_257_Linux_x86_64_cuda_12.9
sle15:~/nvhpc_2025_257_Linux_x86_64_cuda_12.9 # ./install
```

| 步驟                              | 意義                                                     |
| --------------------------------- | -------------------------------------------------------- |
| 解壓對應平台 / CUDA 版本的 tar 包 | 版本需與驅動、GPU 架構相容                               |
| `./install`                       | 互動或腳本化安裝至預設路徑（常見 `/opt/nvidia/hpc_sdk`） |

#### 環境變數設定

```bash
sle15:~ $ export NVARCH=`uname -s`_`uname -m`
sle15:~ $ export NVCOMPILERS=/opt/nvidia/hpc_sdk
sle15:~ $ export MANPATH=$MANPATH:$NVCOMPILERS/$NVARCH/25.7/compilers/man
sle15:~ $ export PATH=$NVCOMPILERS/$NVARCH/25.7/compilers/bin:$PATH

# MPI
sle15:~ $ export PATH=$NVCOMPILERS/$NVARCH/25.7/comm_libs/mpi/bin:$PATH
sle15:~ $ export MANPATH=$MANPATH:$NVCOMPILERS/$NVARCH/25.7/comm_libs/mpi/man

# module
sle15:~ $ export MODULEPATH=$NVCOMPILERS/modulefiles:$MODULEPATH
sle15:~ $ module load nvhpc

sle15:~ $ echo $NVCOMPILERS
```

| 變數 / 動作                                           | 意義                                         |
| ----------------------------------------------------- | -------------------------------------------- |
| `NVARCH`                                              | 如 `Linux_x86_64`，組成安裝路徑              |
| `NVCOMPILERS`                                         | SDK 根目錄                                   |
| 把 `compilers/bin` 與 `comm_libs/mpi/bin` 加入 `PATH` | 找到 `nvc`、`mpirun` 等                      |
| `MODULEPATH` + `module load nvhpc`                    | 用 Lmod / Modules 一次載入（建議用於叢集）   |
| 路徑中的 `25.7`                                       | 對應 SDK 版本，升級時需改版本號或改用 module |

#### 編譯範例

```bash
sle15:~ $ which nvc
sle15:~ $ nvc -o hello hello.c

sle15:~ $ which nvc++
sle15:~ $ nvc++ -o hello hello.cpp

sle15:~ $ which nvfortran
sle15:~ $ nvfortran -o hello -Mfixed hello.f
sle15:~ $ nvfortran -o hello -Mfree hello.f90

sle15:~ $ mpicc   -o mpi_pi mpi_pi.c
sle15:~ $ mpicxx  -o mpi_pi mpi_pi.cpp
sle15:~ $ mpif77  -o mpi_pi mpi_pi.f
sle15:~ $ mpifort -o mpi_pi mpi_pi.f90

sle15:~ $ mpirun -n <NPROC> ./mpi_pi
```

| 編譯器 / 選項                 | 意義                                                       |
| ----------------------------- | ---------------------------------------------------------- |
| `nvc` / `nvc++` / `nvfortran` | NVIDIA HPC 編譯器                                          |
| `-Mfixed` / `-Mfree`          | Fortran 固定 / 自由格式                                    |
| `mpifort`                     | Fortran 90 / 現代 Fortran 的 MPI wrapper（名稱依套件慣例） |

#### NVIDIA 工具鏈建議步驟

1. 確認 GPU 驅動與 CUDA 版本相容
2. 安裝 HPC SDK
3. 設定 `PATH` 或 `module load nvhpc`
4. 編譯序列與 MPI hello
5. 若要用 GPU，再以裝置查詢工具確認可見 GPU 後編譯加速選項

### 學習心得：NVIDIA HPC SDK

- CPU 科學碼與 GPU 加速碼的工具鏈選擇可能不同；同一叢集常用 Lmod 並存 Intel 與 NVIDIA 兩套。
- 版本目錄（如 `25.7`）寫死在 `export` 裡不易維護，正式環境應 module 化。
- MPI 啟動方式仍應與資源管理器（Slurm）整合，避免在登入節點直接佔用。

### 練習：NVIDIA HPC SDK

1. 說明 PGI 與 NVIDIA HPC SDK 的關係。
2. 解釋 `NVARCH`、`NVCOMPILERS` 如何組成編譯器路徑。
3. 比較 `nvfortran -Mfixed` 與 Intel `ifx -fixed` 的目的是否相同。

---

## 6-2. Chemistry Computer Program（化學 / 材料計算軟體）

以下軟體多屬授權或學術授權管控，安裝前需確認授權條款、授權伺服器與允許的編譯器組合。

### 6-2-1. VASP

官網：[VASP](https://www.vasp.at/)

#### 歷史與定位

**VASP**（Vienna Ab initio Simulation Package）源自維也納大學團隊發展的平面波 / PAW 第一原理計算套件，廣泛用於固態物理、材料科學與表面化學等。特色包括：

- 平面波基底與 PAW 方法
- 週期系統電子結構、結構優化、分子動力學等
- 高度仰賴高效 Fortran / MPI（近年亦有 GPU 相關建置目標）

授權通常需向官方取得原始碼與勢函數（POTCAR）使用權，不可任意散佈。

#### 建置前置

```bash
sle15:~ # # 確認 Intel oneAPI 環境已載入
sle15:~ # which icx icpx mpiifx
```

| 檢查                      | 意義                                  |
| ------------------------- | ------------------------------------- |
| `icx` / `icpx` / `mpiifx` | 對應 C / C++ / Fortran MPI 工具鏈可用 |

#### 解壓與選用架構 makefile

```bash
sle15:~ # tar zxf vasp.6.4.1.tgz -C /usr/local
sle15:~ # cd /usr/local/vasp.6.4.1
sle15:/usr/local/vasp.6.4.1 # cp arch/makefile.include.intel ./makefile.include
sle15:/usr/local/vasp.6.4.1 # vi makefile.include
sle15:/usr/local/vasp.6.4.1 # vi parse/makefile
```

| 步驟                          | 意義                                                              |
| ----------------------------- | ----------------------------------------------------------------- |
| 解壓到 `/usr/local`           | 集中放置版本目錄（亦可改 `/opt`）                                 |
| 複製 `makefile.include.intel` | 以 Intel 工具鏈為模板                                             |
| 編輯 `makefile.include`       | 把舊 `mpiifort` / `icc` / `icpc` 改成新 `mpiifx` / `icx` / `icpx` |
| 編輯 `parse/makefile`         | parser 測試目標同樣改用 `ifx`                                     |

#### 編譯器名稱遷移（重點）

```makefile
# arch/makefile.include.intel（概念對照）
# 舊：
# FC = mpiifort
# CC_LIB = icc
# CXX_PARS = icpc
# 新：
FC         = mpiifx [-static-intel|-Bstatic]
FCL        = mpiifx -qmkl=sequential [-static-intel|-Bstatic]
CC_LIB     = icx
CXX_PARS   = icpx
```

```makefile
# parse/makefile（概念對照）
# 舊：ifort ...
# 新：ifx ...
```

| 變更                        | 意義                                       |
| --------------------------- | ------------------------------------------ |
| `mpiifort` → `mpiifx`       | 使用 oneAPI Fortran MPI wrapper            |
| `icc`/`icpc` → `icx`/`icpx` | 使用 LLVM 新前端                           |
| `-qmkl=sequential`          | 連結 Intel MKL（序列介面），常見於數值核心 |

#### 編譯目標

```bash
sle15:/usr/local/vasp.6.4.1 # make <target>
# target: std | gam | ncl | gpu | ...

sle15:/usr/local/vasp.6.4.1 # make std
sle15:/usr/local/vasp.6.4.1 # ls bin
```

| target（常見） | 意義                           |
| -------------- | ------------------------------ |
| `std`          | 標準版執行檔（最常用起點）     |
| `gam`          | Gamma-only 等特定用途變體      |
| `ncl`          | non-collinear / 自旋相關變體   |
| `gpu`          | GPU 建置（需對應工具鏈與硬體） |

- 成功後於 `bin/` 可見如 `vasp_std` 等執行檔。

#### 測試計算（NaCl 範例）

```bash
sle15:~ $ cp -r /usr/local/vasp.6.4.1/testsuite/tests/NaCl .
sle15:~ $ cp /usr/local/vasp.6.4.1/testsuite/POTCARS/POTCAR.NaCl NaCl/POTCAR
sle15:~ $ cp /usr/local/vasp.6.4.1/bin/vasp_std NaCl/
sle15:~ $ cd NaCl
sle15:~/NaCl $ mpirun -np 2 ./vasp_std
```

| 步驟            | 意義                                 |
| --------------- | ------------------------------------ |
| 複製測試目錄    | 取得 INCAR / POSCAR / KPOINTS 等輸入 |
| 準備 `POTCAR`   | 勢函數；無授權勢檔無法正式計算       |
| 執行 `vasp_std` | 以 2 行程驗證建置可用                |

#### VASP 建議步驟總覽

1. 載入 Intel oneAPI
2. 解壓原始碼並複製 intel makefile 模板
3. 將編譯器名稱遷移到 `icx` / `icpx` / `mpiifx` / `ifx`
4. `make std` 並確認 `bin/`
5. 用 testsuite 小算例驗證
6. （叢集）做成 module，並以 Slurm 提交平行作業

### 學習心得：VASP

- 授權、POTCAR、編譯器版本三者缺一不可。
- oneAPI 世代轉換是目前建置文件最容易過時的地方。
- 先求 `std` 通過測試，再編譯 `gam` / `ncl` / `gpu`。

### 練習：VASP

1. 說明為何要把 `mpiifort` 改成 `mpiifx`。
2. 列出 `make std` 成功後應檢查的目錄與檔名。
3. 描述一次最小驗證計算需要哪些輸入檔類型（概念）。

---

### 6-2-2. Gaussian

官網：[Gaussian](https://gaussian.com/)

#### 歷史與定位

**Gaussian** 是量子化學領域最知名的商業套件之一，名稱與高斯型基組（Gaussian-type orbitals）傳統密切相關。由 John Pople 等人開創的方法學與軟體路線，對分子體系的電子結構計算影響深遠（Hartree–Fock、DFT、後 HF 等）。廣泛用於分子性質、反應路徑、光譜相關計算等。

- 以商業授權為主，安裝媒體與建置腳本依版本（如 g16）提供。
- 傳統環境常涉及 `csh` / `tcsh` 腳本與特定編譯器旗標。

#### 系統依賴

```bash
sle15:~ # zypper in -t pattern devel_basis
sle15:~ # zypper in tcsh
sle15:~ # zypper in glibc-devel-32bit
```

| 套件 / pattern      | 意義                                  |
| ------------------- | ------------------------------------- |
| `devel_basis`       | 基礎編譯工具鏈與標頭                  |
| `tcsh`              | 部分 Gaussian 腳本假設 csh 系 shell   |
| `glibc-devel-32bit` | 若建置 / 執行仍依賴 32-bit 元件時需要 |

#### 解壓與權限

```bash
sle15:~ # tar jxf wkssrc.tbz -C /usr/local
sle15:~ # chown -R <user>:<group> /usr/local/g16
```

| 步驟              | 意義                                               |
| ----------------- | -------------------------------------------------- |
| 解壓 `wkssrc.tbz` | 展開 g16 原始 / 工作站原始碼樹（實際檔名依發行物） |
| `chown`           | 讓建置使用者對目錄可寫                             |

#### 載入官方環境腳本

```bash
sle15:~ # export g16root=/usr/local
sle15:/usr/local/g16 # source bsd/g16.profile   # sh / bash
sle15:/usr/local/g16 # source bsd/g16.login     # csh / tcsh
```

| 檔案          | 意義                            |
| ------------- | ------------------------------- |
| `g16.profile` | bash / sh 用環境                |
| `g16.login`   | csh / tcsh 用環境               |
| `g16root`     | Gaussian 安裝根的上一層慣例變數 |

#### 編譯器與架構旗標調整

```bash
sle15:~ # pgf77 -help -tp
sle15:/usr/local/g16 # sed -i s/p7-64/x86-64-v3/ bsd/set-mflags
sle15:/usr/local/g16 # sed -i s/p7-64/x86-64-v3/ bsd/setup-make
sle15:/usr/local/g16 # bsd/bldg16
```

| 步驟                          | 意義                                                    |
| ----------------------------- | ------------------------------------------------------- |
| 檢查 `pgf77 -tp`              | 了解 PGI / NVIDIA 編譯器支援的目標架構選項              |
| 將 `p7-64` 取代為 `x86-64-v3` | 讓 makefile 旗標符合較新 CPU ISA 層級（依實際機器調整） |
| `bsd/bldg16`                  | 官方建置入口腳本                                        |

- 原文若出現 `set -i`，應為 **`sed -i`** 就地取代。

#### 執行環境與測試

```bash
sle15:~ # export g16root=/usr/local
sle15:~ # export GAUSS_EXEDIR=$g16root/g16:$g16root/g16/bsd
sle15:~ # export GAUSS_SCRDIR=/tmp
sle15:~ # export LD_LIBRARY_PATH=$GAUSS_EXEDIR:$LD_LIBRARY_PATH
sle15:~ # export PATH=$GAUSS_EXEDIR:$PATH

sle15:~ # mkdir test && cd test
sle15:~/test # cp $g16root/g16/tests/com/test0000.com .
sle15:~/test # g16 test0000.com
```

| 變數            | 意義                                                        |
| --------------- | ----------------------------------------------------------- |
| `GAUSS_EXEDIR`  | 執行檔與相關工具路徑                                        |
| `GAUSS_SCRDIR`  | scratch 目錄；大型計算應指向高速大容量空間，而非填滿 `/tmp` |
| `g16 input.com` | 讀取輸入檔執行計算                                          |

### 學習心得：Gaussian

- Shell 種類（bash / tcsh）與環境腳本要成對，否則變數未載入會找到舊路徑。
- Scratch 位置往往比 CPU 更快成為瓶頸或故障點，需在作業腳本中明確指定。
- 架構旗標（`-tp` / `x86-64-v3`）必須匹配實際節點 CPU，盲目複製文件易導致非法指令。

### 練習：Gaussian

1. 說明 `g16root`、`GAUSS_EXEDIR`、`GAUSS_SCRDIR` 各自控制什麼。
2. 為何叢集環境不建議長期把 `GAUSS_SCRDIR` 指到小型 `/tmp`？
3. 解釋修改 `bsd/set-mflags` 的目的。

---

### 6-2-3. GROMACS

官網：[GROMACS](https://www.gromacs.org/)

#### 歷史與定位

**GROMACS**（GROningen MAchine for Chemical Simulations）起源於格羅寧根大學的分子動力學（MD）計畫，後來發展為廣泛使用的開源 MD 套件。特色包括：

- 生物分子（蛋白質、脂質、核酸等）模擬常見選擇
- 高度最佳化的 CPU 核心，並積極支援 GPU 加速
- 以 CMake 建置為主，可搭配 GCC、Intel、NVIDIA 等編譯器
- 授權相對友善（開源），適合教學與研究叢集廣佈

#### 建置概念步驟（實務指引）

1. 載入選定編譯器與 MPI（Lmod）
2. 安裝 / 啟用 CMake、FFT 函式庫（如 FFTW）、可選 CUDA
3. out-of-source 建置（`cmake` + `make` / `ninja`）
4. `make check` 或官方回歸測試
5. 安裝至版本目錄並撰寫 modulefile
6. 以 `gmx` / `gmx_mpi` 跑短模擬驗證

| 注意點             | 意義                               |
| ------------------ | ---------------------------------- |
| MPI 與 thread 並行 | 要與 Slurm 配置的 CPU 綁定策略一致 |
| GPU 建置           | 需匹配 CUDA 與驅動                 |
| 單精 / 雙精        | 不同研究需求與效能取捨             |

### 學習心得：GROMACS

- 開源使其成為練習「編譯器 × MPI × GPU × module」整合的良好標的。
- 效能高度依賴編譯選項與硬體綁定，不是裝完就能代表峰值效能。

### 練習：GROMACS

1. 說明 GROMACS 與 VASP / Gaussian 在授權模式上的差異。
2. 列出建置前應載入的三類元件：編譯器、MPI、數學 / FFT 函式庫。

---

### 6-2-4. Quantum ESPRESSO（QE）

官網：[Quantum ESPRESSO](https://www.quantum-espresso.org/)

#### 歷史與定位

**Quantum ESPRESSO**（常簡稱 QE）是開源的平面波 / 赝勢電子結構套件，名稱中的 ESPRESSO 代表開源套件集合精神（opEn-Source Package for Research in Electronic Structure, Simulation, and Optimization）。社群來自義大利與國際合作，廣泛用於：

- DFT 電子結構
- 結構優化、聲子、分子動力學等（視編譯的套件組件）
- 教學與研究上替代 / 互補商業或授權型平面波碼

常見執行檔包括 `pw.x`（平面波自洽計算）等；建置多以 Fortran 編譯器 + MPI + 數學庫（BLAS / LAPACK / ScaLAPACK / FFTW）完成。

#### 建置概念步驟（實務指引）

1. 載入 Fortran / C 編譯器與 MPI
2. 準備數學庫（可為 openblas、MKL 等）
3. `./configure` 或 CMake（依版本文件）偵測環境
4. `make all`（或依需要選组件）
5. 將 `bin/` 納入 module
6. 以官方 example 跑 `pw.x` 小算例驗證

| 項目         | 意義                                         |
| ------------ | -------------------------------------------- |
| 赝勢檔       | 類似其他平面波碼，需準備合適 pseudopotential |
| 平行策略     | k 點 / 平面波 / 池化等平行選項影響擴展性     |
| 與 VASP 對照 | 同屬第一原理平面波路線，但授權與輸入生態不同 |

### 學習心得：Quantum ESPRESSO

- 開源平面波生態適合驗證工具鏈是否完整（編譯器、MPI、數學庫）。
- 輸入與赝勢管理同樣需要版本化與文件化，否則結果難以重現。

### 練習：Quantum ESPRESSO

1. 比較 QE 與 VASP 在授權與典型使用情境上的異同。
2. 說明為何科學計算叢集常同時提供 Intel 與 GCC / NVIDIA 多套 module。

---

## 步驟總結

科學計算環境落地建議順序：

1. **選定並安裝工具鏈**: Intel oneAPI 與 / 或 NVIDIA HPC SDK
2. **環境模組化**: 用 Lmod 包裝 `oneapi-vars` / `nvhpc`，避免手寫 `export` 漂移
3. **驗證編譯器**: `hello.c` / `.f90` → MPI 範例 → `mpirun` / `srun`
4. **依授權安裝領域軟體**:
   - VASP：遷移 makefile 至 `icx` / `ifx` / `mpiifx`，`make std`，跑 testsuite
   - Gaussian：依賴套件 → 解壓 → 調整架構旗標 → `bldg16` → 設定 `GAUSS_*`
   - GROMACS / QE：依官方建置系統編譯、測試、安裝
5. **接到資源管理器**: 以 Slurm 提交，明確指定模組、行程數、scratch 目錄
6. **文件化版本組合**: 編譯器版本 × MPI × 應用版本，確保計算可重現

---

## 總結

本章把科學計算所需的「工具鏈」與「應用層」串起來：Intel oneAPI 與 NVIDIA HPC SDK 分別代表 CPU 最佳化與 GPU / 加速器路線的主流選擇；VASP、Gaussian、GROMACS、Quantum ESPRESSO 則覆蓋授權型與開源型的量子化學 / 材料 / 分子動力學需求。實務成敗關鍵往往不在單一指令，而在**版本一致、環境可載入、makefile 跟隨編譯器世代、測試算例先通過、再交給佇列系統批次執行**。完成後，叢集即可提供可重現的科學計算軟體棧，供教學與研究作業使用。
