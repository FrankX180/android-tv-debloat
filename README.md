# Android TV 安全清理與加速指引 (AI Agent 專用 SOP)

> 本文件專為 **AI Coding Agent（如 Grok CLI / Claude Code / Cursor / Windsurf 等）** 與使用者協同作業設計。  
> 只要使用者在終端中將本專案的 GitHub 網址貼給 AI，AI 即可自動讀取本 SOP，引導使用者完成電視前置設定，並透過 ADB 自動化連線、安全分批停用無用廣告/垃圾套件、縮短過渡動畫，完成電視瘦身加速。

---

## 🎯 給使用者的第一步：電視前置設定

> ⚠️ **必要網路條件（最重要）**：  
> **電視** 與 **執行 AI 的電腦** **必須連線在「同一個 Wi-Fi」或「同一個區域網路（同一台路由器/分享器）」下**，否則電腦將無法透過 IP 連線至電視！

在讓 AI 連線之前，請先拿起電視遙控器完成以下 3 個步驟：

1. **開啟開發者模式**：
   - 進入電視：**設定 → 系統 → 關於**。
   - 在「**版本號碼 (Android TV 作業系統版本 / Build)**」上連續按 **7 次 OK 鍵**，直到畫面提示「您現在已是開發人員」。
2. **開啟 USB 偵錯**：
   - 進入電視：**設定 → 系統 → 開發者選項**。
   - 開啟 **USB 偵錯**（若電視有顯示「無線偵錯」，亦可開啟）。
3. **確認同 Wi-Fi 並查詢電視 IP**：
   - 確認電視與電腦連線在同一個 Wi-Fi 名稱（SSID）下。
   - 進入電視：**設定 → 網路與網際網路 → 目前連線的 Wi-Fi / 有線網路**。
   - 記下電視的 IP 位址（例如 `192.168.1.50` 或 `192.168.68.78`）。
4. **將 IP 貼給 AI**：
   - 將 IP 發在對話框中，告訴 AI：「我的電視 IP 是 xxx，請幫我開始清理加速」。

---

## 🤖 給 AI Agent 的執行規範與標準 SOP

### 一、 核心安全鐵律 (Zero-Brick Rules)
1. **嚴禁直接解除安裝 (`pm uninstall`)**：
   - 一律僅使用 `pm disable-user --user 0 <package>`，確保隨時可透過 `pm enable <package>` 100% 完整還原。
2. **嚴禁 Root / 解鎖 Bootloader**：
   - 避免觸發 Widevine 降級（Widevine L1 降至 L3 會導致 Netflix / Disney+ / Prime Video 只能播放 480p/720p 標清畫質）。
3. **基準量測與備份**：
   - 執行停用前，必須建立 `tv-debloat/` 工作目錄，備份原始記憶體基準 (`dumpsys meminfo`) 與完整套件清單。
4. **小批次執行與使用者驗證（每批 ≤ 10 個）**：
   - 每停用一批，必須請使用者用遙控器測試核心功能（HDMI 訊號源切換、音量、輸入法、常用串流 App）。
5. **落盤紀錄與一鍵還原腳本**：
   - 所有停用項目必須記錄於 `tv-debloat/disabled_packages.txt`，並自動產出 `tv-debloat/restore_all.bat`。

---

### 二、 絕對禁止停用的核心白名單 (Blacklist)

| 套件特徵 / 名稱 | 功能用途 | 誤停用後果 |
|---|---|---|
| `com.tcl.suspension` / 品牌訊號源面板 | 遙控器 Source / Inputs 快速選單 | **無法切換 HDMI 訊號源** |
| `com.tcl.tv`, `com.tcl.tvinput` / 品牌 TV Input | HDMI / 天線訊號底層服務 | 無法顯示外接設備畫面 |
| `com.google.android.tv.remote.service` | 遙控器通訊服務 | 遙控器斷連/失靈 |
| `*.bt_rcu.*`, `*.autopair` | 藍牙遙控器配對與通訊 | 藍牙遙控器失效 |
| `com.google.android.gms`, `com.google.android.gsf` | Google Play 服務框架 | 系統與依賴 App 崩潰 |
| `com.android.vending` | Google Play 商店 | 無法更新或下載 App |
| `com.android.location.fused` | 定位基礎服務 | **直接導致系統無限重開機 (Bootloop)** |
| `*.inputmethod.*` (如 Gboard / LatinIME) | 螢幕虛擬鍵盤 | 無法輸入文字或搜尋 |
| `com.google.android.apps.tv.launcherx` | 官方 Google TV 預設主畫面 | **未安裝替代桌面即停用會導致黑畫面** |

---

### 三、 標準自動化執行步驟 (AI Agent 流程)

#### 步驟 1：ADB 連線與基準記錄
```powershell
# 1. 連線至電視
adb connect <TV_IP>:5555
# 提醒使用者在電視畫面上勾選「永遠允許來自此電腦的偵錯」並按確定

# 2. 建立工作目錄並抓取原始基準
mkdir tv-debloat
adb shell dumpsys meminfo > tv-debloat/meminfo_before.txt
adb shell pm list packages -s > tv-debloat/packages_system_original.txt
adb shell pm list packages -d > tv-debloat/packages_disabled_original.txt
```

#### 步驟 2：套件分析與分類分級
AI 讀取 `packages_system_original.txt`，分析電視廠牌（Sony / TCL / Philips / Xiaomi / Chromecast 等），將套件分成三類並產出表格給使用者確認：
- **安全停用組**：未訂閱的海外串流（如 Starz, Peacock, Hulu, Sling）、零售展示模式 (Retail/Demo mode)、廠商廣告遙測與分析追蹤（Samba TV 等）、語音助理引導精靈。
- **確認停用組**：廠商內建專屬影音中心、客製自帶瀏覽器、專用螢幕保護程式、診斷工具。
- **保留核心組**：白名單與系統基礎框架。

#### 步驟 3：分批停用與驗證 (每批 ≤ 10 個)
```powershell
adb shell pm disable-user --user 0 <package_name>
```
- 每批執行後，請使用者測試：HDMI 切換、音量調整、YouTube / Netflix 播放、虛擬鍵盤輸入。
- 若有異常，立即執行還原：
```powershell
adb shell pm enable <package_name>
```

#### 步驟 4：系統流暢度加速 (動畫與快取微調)
```powershell
# 縮短系統動畫時間至 0.5x（大幅提升 UI 視覺流暢度）
adb shell settings put global window_animation_scale 0.5
adb shell settings put global transition_animation_scale 0.5
adb shell settings put global animator_duration_scale 0.5

# 清理系統快取
adb shell pm trim-caches 4096M
```

#### 步驟 5：重開機與產出一鍵還原腳本
```powershell
adb reboot
```
電視重啟完成後，AI 自動產出 Windows 一鍵還原腳本 `tv-debloat/restore_all.bat`：
```bat
@echo off
chcp 65001 >nul
echo ===================================================
echo 正在還原所有被停用的電視套件...
echo ===================================================
for /f "tokens=*" %%i in (disabled_packages.txt) do (
    echo [ENABLE] %%i
    adb shell pm enable %%i
)
echo ===================================================
echo 還原完成！所有停用套件已全部重新啟用。
echo ===================================================
pause
```
並交付清理前後 RAM 佔用對照表與已停用套件清單。

---

## 📄 授權條款
MIT License - 歡迎自由轉載、分享與串接至各類 AI Agent 工具中。
