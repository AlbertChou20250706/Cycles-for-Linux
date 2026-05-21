# Cycles-for-Linux
完整部署包 (RHEL 10 + TR-14 + 多類型測試)
# 🎯 Albert SIT v6.0 — 完整部署包 (RHEL 10 + TR-14 + 多類型測試)

## 📦 完整檔案結構

```
~/sit_test/
├── regression_test.sh              ← v5.3 協調器 (環境檢查 + 5min 間隔)
├── Albert_Ultimate_Cycle_Test.sh   ← v6.0 引擎 (TR-14 真實 AC)
├── check_environment.sh            ← 🆕 獨立環境檢查工具
├── stop_reboot_test.sh             ← 緊急停止
├── ver.sh                          ← 配置工具 (沿用舊版)
├── install.sh                      ← 安裝工具 (沿用舊版)
├── tools/                          ← 🆕 工具目錄
│   └── tr-14                       ← TR-14 二進位執行檔
├── config/                         ← 配置檔案 (執行 ver.sh 後產生)
│   ├── ipmi.conf                   ← BMC 連線資訊
│   ├── reboot_test, dc_test, ac_test, shutdown_test  ← 測試啟用旗標
│   └── ...
└── result/                         ← 測試結果 (自動產生)
```

---

## 🆕 v6.0 新功能總覽

| 功能 | 說明 | 涉及檔案 |
|------|------|---------|
| 🔌 **真實 AC 切電** | 透過 `./tools/tr-14` 控制 TR-14 硬體切真實 AC 電源 | engine v6.0 |
| ⏱️ **類型間隔等待** | REBOOT→AC→DC 之間自動等 5 分鐘 (可調整) | regression v5.3 |
| 🔍 **環境檢查** | 缺 package/tool 立即提示工程師 | regression v5.3 + check_environment.sh |
| 📁 **相對路徑** | tr-14 不再寫死 `/usr/bin/`,改用 `./tools/tr-14` | engine v6.0 |
| 🎛️ **互動式設定** | 啟動時可調整間隔時間 | regression v5.3 |

---

## ⚡ 3 步快速部署

### Step 1: 部署檔案

```bash
cd ~/sit_test

# 備份舊版本
mkdir -p backup_$(date +%Y%m%d)
cp regression_test.sh backup_$(date +%Y%m%d)/ 2>/dev/null || true
cp Albert_Ultimate_Cycle_Test.sh backup_$(date +%Y%m%d)/ 2>/dev/null || true

# 部署新版 (從 v6.0 outputs 複製)
# regression_test.sh (v5.3)
# Albert_Ultimate_Cycle_Test.sh (v6.0)
# check_environment.sh (新工具)

# 建立 tools 目錄並放入 tr-14
mkdir -p tools/
# 將 tr-14 複製進來
cp /path/to/tr-14 tools/
chmod 755 tools/tr-14
chmod 755 *.sh

# 驗證版本
grep SCRIPT_VERSION regression_test.sh        # 5.3
grep SCRIPT_VERSION Albert_Ultimate_Cycle_Test.sh  # 6.0
ls -la tools/tr-14                            # 可執行
```

### Step 2: 環境檢查 (新功能)

```bash
# 執行環境檢查
bash check_environment.sh
```

**預期輸出:**
- ✅ 所有核心工具齊全
- ⚠️ 若缺 ipmitool/dmidecode/expect → 顯示安裝指令
- ✅ TR-14 工具就位
- ✅ Serial port 偵測

若有錯誤,工具會列出修復指令,例如:
```
💡 修復建議 (按優先順序):
    1. dnf install -y ipmitool dmidecode pciutils
    2. chmod 600 config/ipmi.conf
    3. 將 tr-14 二進位檔複製到: ...
```

### Step 3: 啟動測試 (含互動式設定)

```bash
# 1. 配置測試項目
sudo bash ver.sh

# 選擇要跑哪些測試:
# 👉 執行 Reboot 測試?    (y/N): y  循環次數: 10
# 👉 執行 AC Cycle 測試?  (y/N): y  循環次數: 5  ← 將用 TR-14 真實 AC
# 👉 執行 DC Cycle 測試?  (y/N): y  循環次數: 5
# 👉 執行 Shutdown 測試?  (y/N): N

# 2. 啟動測試
sudo bash regression_test.sh
```

**啟動時會看到:**
```
[INFO] 🔍 環境檢查 (Preflight Check)
[1/5] 檢查核心 Linux 工具...
[PASS] ✓ 核心 Linux 工具齊全
[2/5] 檢查 systemd 環境...
[PASS] ✓ systemd 可用 (autostart 將使用 systemd)
[3/5] 檢查測試工具...
[PASS] ✓ ipmitool 已安裝
[PASS] ✓ config/ipmi.conf 已配置
[PASS] ✓ rtcwake 已安裝
[4/5] 檢查 TR-14 切電工具...
[PASS] ✓ TR-14 工具存在: tools/tr-14
[PASS] ✓ Serial port 可用: /dev/ttyS0
[5/5] 檢查工作環境...
[PASS] ✓ /root 磁碟空間充足

[INFO] 📊 環境檢查結果
[PASS] ✅ 環境完美!可以開始測試

# 然後會問:
[INFO] 偵測到多個測試類型 (共 3 項)
[INFO] 📋 測試類型間隔設定
   目前設定: 300 秒 (5 分鐘)
👉 是否要修改間隔時間? (y/N): 
```

---

## 🔄 完整測試流程 (REBOOT + AC + DC)

```
時間軸:
00:00 啟動 regression_test.sh
00:01 環境檢查通過
00:02 詢問間隔時間 (預設 5 min)
00:03 開始 REBOOT × 10 圈
      ↓ (每圈約 5 min)
00:53 REBOOT 完成
      ↓
00:53 ⏸️ 等待 5 分鐘 (給 REBOOT 收尾, AC 準備)
00:58 開始 AC × 5 圈 (使用 TR-14 真實 AC)
      ↓ (每圈約 5 min)
01:23 AC 完成
      ↓
01:23 ⏸️ 等待 5 分鐘 (給 AC 收尾, DC 準備)
01:28 開始 DC × 5 圈
      ↓ (每圈約 5 min)
01:53 DC 完成
01:54 🎉 全部測試完成,打包結果
```

---

## 📋 工程師快速參考

### 環境檢查 (任何時候都可執行)
```bash
bash check_environment.sh
```

### 緊急停止
```bash
sudo bash stop_reboot_test.sh
```

### 查看當前狀態
```bash
sudo bash regression_test.sh --status
```

### 中止測試
```bash
sudo bash regression_test.sh --abort
```

### 查看即時日誌
```bash
tail -f /root/Documents/bootstrap.log
```

---

## 🔧 TR-14 部署方法

### 1️⃣ 取得 tr-14 二進位檔

```bash
# 如果之前在 /usr/bin/tr-14
cp /usr/bin/tr-14 ~/sit_test/tools/

# 或從備份取得
cp /path/to/sit_module/tr-14 ~/sit_test/tools/
```

### 2️⃣ 設定權限

```bash
mkdir -p ~/sit_test/tools/
cp tr-14 ~/sit_test/tools/
chmod 755 ~/sit_test/tools/tr-14
```

### 3️⃣ 連接硬體

- 將 TR-14 切電器透過 RS-232 或 USB-Serial 連到主機
- 確認 `/dev/ttyS0` 或 `/dev/ttyUSB0` 出現
- 設定 BMC `chassis policy always-on`

### 4️⃣ 測試

```bash
# 驗證
bash check_environment.sh

# 預期看到:
# ✓ TR-14 工具存在: tools/tr-14
# ✓ Serial port 可用: /dev/ttyS0 (或 /dev/ttyUSB0)
```

---

## ⚙️ 進階設定

### 修改間隔時間

**方法 1: 啟動時互動式修改**
```bash
sudo bash regression_test.sh
# 會問: 👉 是否要修改間隔時間? (y/N): y
# 然後輸入新的秒數
```

**方法 2: 修改原始碼**
```bash
# 編輯 regression_test.sh
# 找到這行:
INTER_TEST_DELAY=300
# 改成你想要的秒數 (60-1800)
```

### 修改 TR-14 切電時間

```bash
# 編輯 Albert_Ultimate_Cycle_Test.sh
# 找到這行:
readonly TR14_OFF_SECONDS=50
# 改成你想要的秒數 (預設 50 秒)
```

### 修改 Serial Port 偵測順序

```bash
# 編輯 Albert_Ultimate_Cycle_Test.sh
# 找到這行:
readonly TR14_SERIAL_PORTS=("/dev/ttyS0" "/dev/ttyUSB0")
# 改成你的順序,例如:
readonly TR14_SERIAL_PORTS=("/dev/ttyUSB0" "/dev/ttyS0" "/dev/ttyS1")
```

---

## 🆘 常見問題

### Q1: AC 測試降級到 IPMI?

**症狀:**
```
[WARN] [AC] ⚠️  TR-14 工具不存在或無執行權限
[WARN] [AC] 將降級使用 IPMI poweroff (DC 模式,非真實 AC)
```

**解決:**
```bash
# 1. 檢查 tr-14 是否存在
ls -la ~/sit_test/tools/tr-14

# 2. 若不存在,放進來
cp /path/to/tr-14 ~/sit_test/tools/
chmod 755 ~/sit_test/tools/tr-14

# 3. 檢查 serial port
ls -la /dev/ttyS* /dev/ttyUSB*

# 4. 重新測試
bash check_environment.sh
```

### Q2: Serial port 權限問題

**症狀:**
```
[WARN] [AC] /dev/ttyS0 存在但無讀寫權限
```

**解決:**
```bash
# 加入 dialout 群組
sudo usermod -aG dialout root

# 或直接改權限
sudo chmod 666 /dev/ttyS0

# 確認
ls -la /dev/ttyS0
```

### Q3: 間隔時間不夠 (5 分鐘太短)?

**解決:** 啟動時選擇修改:
```bash
sudo bash regression_test.sh
# 👉 是否要修改間隔時間? (y/N): y
# 👉 請輸入新的間隔秒數: 600    # 10 分鐘
```

或修改 `INTER_TEST_DELAY` 常數。

### Q4: 環境檢查發現缺 package?

**解決:** 工具會直接告訴你執行什麼:
```bash
[INFO] 💡 修復建議:
    1. dnf install -y ipmitool dmidecode pciutils
    2. dnf install -y expect zip
```

複製貼上執行即可。

### Q5: 想跳過環境檢查?

**不建議**,但可以:
```bash
# 修改 regression_test.sh,註解掉:
# preflight_check
```

---

## 📊 版本對照表

| 元件 | 舊版本 | v6.0 版本 | 改進 |
|------|--------|----------|------|
| **regression_test.sh** | v5.0 → v5.2 | **v5.3** | + 間隔等待 + 環境檢查 |
| **Albert_Ultimate_Cycle_Test.sh** | v5.8 → v5.9 | **v6.0** | + TR-14 真實 AC + 相對路徑 |
| **AC 切電方法** | IPMI poweroff (假 AC) | **TR-14 硬體 (真 AC)** | 真實切斷 AC |
| **環境檢查** | 無 | **完整檢查** | 缺料立即提示 |
| **多類型間隔** | 無 (緊接) | **5 分鐘 (可調)** | 給 log 時間 |
| **tr-14 路徑** | `/usr/bin/tr-14` (絕對) | **`./tools/tr-14` (相對)** | 可攜性 |

---

## 🎯 部署檢查清單

部署 v6.0 後,工程師應確認:

- [ ] regression_test.sh 版本為 5.3
- [ ] Albert_Ultimate_Cycle_Test.sh 版本為 6.0
- [ ] check_environment.sh 已部署
- [ ] tools/tr-14 已部署且權限為 755
- [ ] 執行 `bash check_environment.sh` 全部通過
- [ ] config/ipmi.conf 已配置 (AC/DC 測試需要)
- [ ] BMC Power Restore Policy = always-on
- [ ] BIOS 已啟用 Dynamic Time / RTC Wake
- [ ] Serial port 可讀寫 (TR-14 連線)
- [ ] 第一次啟動測試時看到「環境完美」訊息
- [ ] 第一次測試類型切換時看到「等待 5 分鐘」

---

## 💡 工程師最佳實踐

### 開發階段
```bash
# 用短時間驗證
sudo bash ver.sh
# REBOOT × 2, AC × 2, DC × 2
sudo bash regression_test.sh
# 互動時設定間隔: 60 (1分鐘) 加快測試
```

### 量產階段
```bash
# 完整壓力測試
sudo bash ver.sh
# REBOOT × 100, AC × 50, DC × 50
sudo bash regression_test.sh
# 間隔保持 300 (5分鐘) 或更長
```

### 監控階段
```bash
# 終端 1: 主測試
sudo bash regression_test.sh

# 終端 2: 監控
tail -f /root/Documents/bootstrap.log

# 終端 3: 環境健康
watch -n 60 'bash check_environment.sh | tail -10'
```

---

## ✨ 最後一句話

**v6.0 版本是工程師友善的版本:**
- 缺什麼,立即告訴你 ✅
- 真實 AC 切電 (不再假裝) ✅
- log 收集時間充裕 ✅
- 一切相對路徑,可攜性高 ✅

**祝測試順利!** 🚀

---

最後更新: 2026-05-15
作者: Albert Chou (志偉 周)
環境: RHEL 10.x
