# Android TV 安全清理與加速手冊 (ADB Debloat)

> **位置**：`E:\_PluginTools\ComputerUSE\01_Docs\Android_TV_Debloat.md`  
> **用途**：隨選啟用（On-Demand）。平常不常駐於系統 Skill 清單以節省 Token，需要清理電視時直接由 AI 讀取本檔執行。

---

## 一、 核心安全鐵律 (Zero-Brick Rules)

1. **嚴禁解除安裝 (`pm uninstall`)**：一律僅使用 `pm disable-user --user 0 <package>`，保證隨時可用 `pm enable <package>` 100% 完整還原。
2. **嚴禁 Root / 解鎖 Bootloader**：避免觸發 Widevine 降級（L1 掉至 L3 會導致 Netflix/Disney+ 只能跑 480p/720p SD 畫質）。
3. **先量測基準**：執行前必須記錄基準記憶體 (`dumpsys meminfo`) 與完整套件清單。
4. **小批次執行（每批 ≤ 10 個）**：每停用一批，必須請使用者用遙控器測試核心功能（訊號源/HDMI 切換、聲音、輸入法、常用 App）。
5. **落盤紀錄與一鍵還原**：所有停用項目必須寫入 `tv-debloat/disabled_packages.txt`，並產生 `tv-debloat/restore_all.bat` 一鍵還原腳本。

---

## 二、 絕對禁止停用的核心白名單 (Blacklist for Disabling)

| 套件特徵 / 名稱 | 功能用途 | 誤停用後果 |
|----------------|---------|-----------|
| `com.tcl.suspension` (或各品牌訊號源面板) | 遙控器 Source / Inputs 快速選單 | **無法切換 HDMI 訊號源** |
| `com.tcl.tv`, `com.tcl.tvinput` (或品牌 TV Input) | HDMI / 天線訊號底層服務 | 無法顯示外接設備畫面 |
| `com.google.android.tv.remote.service` | 遙控器通訊服務 | 遙控器斷連/失靈 |
| `com.tcl.tcl_bt_rcu_service`, `com.tcl.autopair` | 藍牙遙控器配對與通訊 | 藍牙遙控器失效 |
| `com.google.android.gms`, `com.google.android.gsf` | Google Play 服務框架 | 系統與依賴 App 崩潰 |
| `com.android.vending` | Google Play 商店 | 無法更新或下載 App |
| `com.android.location.fused` | 定位基礎服務 | **直接導致系統無限重開機 (Bootloop)** |
| `*.inputmethod.*` (如 `com.google.android.inputmethod.latin`) | 螢幕虛擬鍵盤 | 無法輸入文字或搜尋 |
| `com.google.android.apps.tv.launcherx` | 官方 Google TV 預設主畫面 | **未安裝替代桌面即停用會導致黑畫面** |

---

## 三、 標準執行 SOP

### 步驟 1：電視前置設定（使用者操作）
1. 在電視上進入：**設定 → 系統 → 關於**，在「版本號碼 (Build)」連續按 **7 次 OK 鍵** 開啟開發者選項。
2. 進入：**設定 → 系統 → 開發者選項**，開啟 **USB 偵錯**（若有「無線偵錯」亦可開啟）。
3. 進入：**設定 → 網路與網際網路 → 狀態**，記下電視的區網 IP（例如 `192.168.1.50`）。

---

### 步驟 2：ADB 連線與基準量測
1. 檢查本機 ADB 環境：
   ```powershell
   adb version
   ```
2. 連線至電視：
   ```powershell
   adb connect <TV_IP>:5555
   ```
   *提醒使用者在電視螢幕上勾選「永遠允許來自此電腦的偵錯」並按確定。*
3. 建立工作目錄並抓取原始基準：
   ```powershell
   mkdir tv-debloat
   adb shell dumpsys meminfo > tv-debloat/meminfo_before.txt
   adb shell pm list packages -s > tv-debloat/packages_system_original.txt
   adb shell pm list packages -d > tv-debloat/packages_disabled_original.txt
   ```

---

### 步驟 3：套件掃描與分級清單
Agent 讀取 `packages_system_original.txt`，分析並分類為三組產出表格：
- **第一組（確定無用/預載垃圾）**：未使用的海外串流（如 Starz, Peacock, Hulu）、展示模式 (Demo/Retail mode)、廠商遙測追蹤、語音助理引導精靈。
- **第二組（需確認）**：特定品牌內建影音中心、客製瀏覽器、屏保、系統診斷工具。
- **第三組（嚴禁停用）**：上述白名單及核心系統驅動。

---

### 步驟 4：替換主畫面 (可選 / 推薦 FLauncher)
若要徹底移除官方主畫面廣告行：
1. 請使用者在電視 Play Store 下載安裝開源極簡桌面 **FLauncher**。
2. 開啟一次 FLauncher 並設定為預設主畫面。
3. 確認生效後，再停用官方桌面：
   ```powershell
   adb shell pm disable-user --user 0 com.google.android.apps.tv.launcherx
   ```

---

### 步驟 5：分批停用與功能測試
1. 每次停用 ≤ 10 個套件：
   ```powershell
   adb shell pm disable-user --user 0 <package_name>
   ```
2. 將停用項目記錄於 `tv-debloat/disabled_packages.txt`。
3. 每次批次完成，要求使用者測試：
   - 遙控器訊號源鍵 (HDMI 切換)
   - 聲音大小聲與靜音
   - YouTube / Netflix 開啟
   - 螢幕鍵盤輸入
4. 若有異常，立即針對該批次單一或全部還原：
   ```powershell
   adb shell pm enable <package_name>
   ```

---

### 步驟 6：系統微調 (加速 & 清理)
1. 縮短系統動畫時間至 0.5x（顯著提高反應流暢度）：
   ```powershell
   adb shell settings put global window_animation_scale 0.5
   adb shell settings put global transition_animation_scale 0.5
   adb shell settings put global animator_duration_scale 0.5
   ```
2. 清理快取：
   ```powershell
   adb shell pm trim-caches 4096M
   ```

---

### 步驟 7：重啟與結案報告
1. 重啟電視：
   ```powershell
   adb reboot
   ```
2. 重啟連線後記錄瘦身後數據：
   ```powershell
   adb shell dumpsys meminfo > tv-debloat/meminfo_after.txt
   adb shell pm list packages -d > tv-debloat/packages_disabled_final.txt
   ```
3. 產出 `tv-debloat/DEBLOAT-LOG.md` 與 Windows 一鍵還原腳本 `tv-debloat/restore_all.bat`：
   ```bat
   @echo off
   echo 正在還原所有被停用的電視套件...
   for /f "tokens=*" %%i in (disabled_packages.txt) do (
       echo Enabling %%i
       adb shell pm enable %%i
   )
   echo 還原完成！
   pause
   ```
4. 交付前後 RAM 佔用對照表與已停用套件說明清單。
