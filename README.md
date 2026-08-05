# 課架（Course Shelf）

您的線上課程集中管理架 — 手動記錄跨平台線上課程的購買與學習進度。

- **Android**：Google Play 正式版（**2.0.0 (34) 已於 2026-08-01 發布上線**）
- **iOS**：App Store（TestFlight 驗證中）
- **收費**：買斷制 NT$500，7 天免費試用
- **開發者**：無聲有愛工作室

---

## 技術架構

| 項目 | 內容 |
|---|---|
| 框架 | React Native 0.85 + Expo SDK 56 + expo-router |
| 語言 | TypeScript |
| 後端 | Supabase（Seoul 區）— Auth / Postgres / Storage / Edge Functions |
| 登入 | Google、LINE |
| 內購 | react-native-iap v15.3.6（Nitro modules） |
| 建置 | EAS Build（雲端編譯） |
| 套件代號 | `com.amanda141319.courseshelf` |
| IAP 商品 ID | `courseshelf_lifetime` |

---

## 專案結構

```
course-shelf/
├─ app.json                 原生設定（圖示、啟動畫面、權限）
├─ eas.json                 EAS Build 設定
├─ .eas/workflows/
│  ├─ build-android.yml               推 master → preview APK（測試用）
│  └─ build-android-production.yml    推 release → production AAB（送審用）
├─ src/
│  ├─ app/
│  │  ├─ _layout.tsx        expo-router 版面（掛載 AppTabs）
│  │  └─ index.tsx          ★ 主程式：三個分頁、所有彈窗、全部樣式
│  ├─ components/
│  │  └─ app-tabs.tsx       expo-router 分頁容器（隱藏原生 tab bar）
│  └─ lib/
│     └─ supabase.ts        Supabase 用戶端
├─ assets/images/           icon / adaptive-icon / splash-icon / favicon
└─ supabase/
   ├─ config.toml
   └─ functions/
      ├─ delete-account/    刪除帳號（清 auth + 資料 + Storage）
      ├─ line-login/        LINE 登入換 Supabase session
      └─ verify-purchase/   伺服器端驗證 Google Play 購買
```

---

## 各機器路徑

| 機器 | 路徑 | 來源資料夾 |
|---|---|---|
| 家用電腦 | `C:\Users\amand\projects\course-shelf` | `src\` |
| 辦公室電腦 | `C:\Users\User\Desktop\projects\course-shelf` | `src\` |
| 手機 | 直接在 GitHub 網頁編輯，觸發自動 build | — |

> repo 預設分支是 **`master`**（不是 main）。

---

## 常用指令

```bash
npm install              # 安裝相依套件
npx expo start           # 本機開發（需 dev client）
eas build -p android     # 雲端建置 Android
eas build -p ios         # 雲端建置 iOS
```

### 全手機送審流程（2026-08-01 建立）

1. 改好程式 commit 到 **`master`** → 自動觸發 **preview APK**，裝到測試機驗證
2. 驗證通過後開 PR 合併進 release：
   `github.com/amanda1413191111-png/course-shelf/compare/release...master`
   → Create pull request → Merge → 自動觸發 **production AAB**
3. 到 expo.dev 下載 AAB → 上傳 Play Console **內部／封閉測試**（免審核）
4. 測試機透過測試軌道從 Play 商店安裝，實機驗證（含 IAP）
5. 驗證通過後，**同一個版本直接晉升正式版**送審

> ⚠️ AAB 不能直接安裝到手機，實機驗證一律走測試軌道。
> ⚠️ 電腦手動打包的 build 在 expo.dev 顯示的 commit 會帶星號（如 `8c1caa8*`），
> 代表含有未 commit 的本機變更、**無法追溯內容**。未來 production AAB
> 一律使用 release 管線產物，確保送審版與測試版同源可考。

### Edge Function 部署

```bash
npx supabase login                          # 一次性
npx supabase link --project-ref <專案代號>   # 一次性
npx supabase functions deploy verify-purchase   # 每次改完跑這行
npx supabase functions deploy                   # 不指定名稱＝部署全部
```

**repo 為唯一來源**：一律在本機改程式 → commit → 用上面指令部署，
不要直接在 Supabase Dashboard 改，否則兩邊會不同步。

`supabase/config.toml` 內每支函式的 `verify_jwt` 必須與 Dashboard 現況一致，
**未寫入設定檔的函式會套用 CLI 預設值（true），可能默默覆蓋掉 Dashboard 的設定**。
目前設定：line-login `false`、delete-account `false`、verify-purchase `true`。

---

## 開發備忘 — 踩過的坑

### Android 陰影渲染

- **inset `boxShadow` 在 Android 一定畫成方形**，不跟 `borderRadius`。膠囊或大圓角元件千萬別用，會露出方角。
- 立體感改用「**同色柔和亮邊 border + 原生 `elevation`**」，兩者都完整貼合 `borderRadius`。
- **四邊給不同顏色的 border**，在大圓角處會出現生硬的對角接縫（看起來一半白一半灰）。要用就四邊同色。
- `overflow: 'hidden'` **不能和 `elevation`/`shadow*` 放同一層**，會把自己的外陰影裁掉；也會讓子元件的 `borderRadius` 失效。
- `shadowOffset` / `shadowOpacity` 是 iOS 專用，Android 只認 `elevation`。
- **圓角容器必然在角落外側留下缺口**，會露出底層頁面。加深遮罩、裁切都只能減輕不能消除；
  底部彈窗上緣因此改為直角（業主決定）。

### Android 層級（z-order）

**`elevation` 會蓋過 `zIndex`。** 目前的層級約定：

```
導覽列膠囊 / FAB  = 8
課程彈窗 overlay  = 30
自製對話框 overlay = 50
```

新增浮動元件時要對照這張表，否則會蓋住彈窗內容。

### 對話框

原生 `Alert` 樣式不受控，已全面改為自製的 `clayAlert(title, msg, buttons)`，呼叫方式與 `Alert.alert` 相同。

> ⚠️ 直式按鈕容器內**不可用 `flex: 1`**（`flexBasis: 0` 會讓文字高度被壓成 0、整顆按鈕變空白）。要用 `alignSelf: 'stretch'`，只有並排時才加 `flex: 1`。

### 內購（IAP）

- `verify-purchase` Edge Function 會向 Google Play Developer API 對帳，
  `purchaseState` 0=已購買 1=已取消或退款 2=待處理，**非 0 一律拒絕解鎖**（已實測有效）。
- **Play Console 退款表單有「移除授權」核取方塊，但預設不勾**。
  - 勾了 → 乾淨退款，授權撤銷，使用者可全新購買。**開發者退款時務必記得勾。**
  - 忘勾 → 製造**死收據**：Google 仍認定「已擁有」（重買被擋），但收據已失效（還原被拒），
    卡成死結。使用者透過 Google 客服退款也可能走到這條路。
  - 事後補撤銷只能透過 Developer API 的 `orders:refund?revoke=true`。
- **死收據解法（2.0.0 (32) 起內建，已完整實測）**：收尾統一由 `markPremium(purchaseToken, purchase)` 處理 —
  - 驗證通過 → `acknowledge`（isConsumable: false）
  - 伺服器**明確**回 `not_purchased` → **`consume`**（isConsumable: true），
    釋放 Play 的「已擁有」狀態，使用者即可重新購買
  - 其他失敗（網路錯誤、暫時性驗證失敗）→ 一律 `acknowledge` 保住付款
    （**未 acknowledge 的購買 3 天後會被 Google 自動退款**），之後用「還原」補解鎖
  - 監聽器在無 session 時也先 `acknowledge`
- **「你已擁有這項商品」黑底彈窗是 Google Play 原生視窗**，認的是**手機 Play 商店登入的
  Google 帳號**，與 App 內登入的帳號（Google/LINE）完全無關。
- Google 回報 `E_ALREADY_OWNED`（或 responseCode 7）代表該帳號已擁有此商品但 App 尚未解鎖，
  此時應自動改呼叫 `restorePurchase()`，不要把原始英文錯誤丟給使用者。
  若該筆收據其實已退款，還原會跳「購買的訂單已失效」並 consume 死收據，之後可重買。
- `supabase.functions.invoke` 在非 2xx 時 **`data` 為 `null`**，錯誤內容要用
  `await error.context.json()` 取，直接讀 `data.error` 讀不到。
- **退款不會回寫 Supabase**：`user_status.is_premium` 仍為 `true`，App 會持續顯示已解鎖
  （對買斷制可接受；重新驗證時會正確處理）。測試時要重現購買卡片，
  需在 Dashboard 手動把 `is_premium` 改回 `false`。
- 刪除本服務帳號**不會**影響 Google Play 的購買紀錄；使用者以同一個 Google Play 帳號
  重新登入後仍可還原永久版。App 內與法規頁的文案須與此行為一致。

### Supabase

- Edge Function 的 `verify_jwt` 要看**這支函式是否需要接受未登入的請求**：
  - **需要**（如 `line-login`：使用者還沒登入才會呼叫）→ 設 `false`，否則閘道會先擋掉。
  - **不需要**（如 `verify-purchase`：App 端已先確認有 session 才呼叫）→ 設 `true` 較佳，
    閘道先驗一次、函式內再用 `getUser()` 驗一次，等於多一層防線。
  - 兩者皆安全，因為函式內部本來就會自行驗證身分；**改動時務必同步 Dashboard 與 `config.toml`**，
    並立即實測「已購買過？點此還原」。
- `SUPABASE_SERVICE_ROLE_KEY` 由平台自動注入，**不可手動設成 secret**。
- Storage 上傳除了 INSERT，**還需要 `storage.objects` 的 SELECT policy**，否則前置檢查會靜默失敗。
- RLS 必須擋住客戶端寫入 `is_premium`、`purchased_at`、`purchase_token`，這些只能由 Edge Function 寫。
- Storage policy 一律綁定使用者資料夾，切勿對 `{public}` 開放：

  ```sql
  bucket_id = 'course-images'
  and (storage.foldername(name))[1] = auth.uid()::text
  ```

  上傳路徑格式為 `{user_id}/{時間戳}_{亂數}.{副檔名}`，第一層資料夾即為使用者 ID。
  `anon key` 打包在 APK 內可被反編譯取出，若 policy 對 `{public}` 開放，
  任何人都能列舉、下載、甚至刪除全部使用者的檔案。
- `delete-account` Edge Function 使用 `SERVICE_ROLE`，會繞過 RLS，不受上述 policy 影響。

### EAS / Expo

- `versionCode` 由 EAS 遠端管理並自動遞增，**不要寫死在 `app.json`**。
- slug 必須用連字號且與雲端註冊完全一致。
- `react-native-svg` v15.15.4 與 SDK 56 / 新架構（Fabric）相容。
- 改動 `app.json` 或 `assets/images/` 屬於原生設定，**必須重新 build**，OTA 更新吃不到。
- production AAB 由 `.eas/workflows/build-android-production.yml` 監聽 **`release` 分支**自動打包；
  平常推 `master` 只會產 preview APK，不消耗 production build。

---

## 安全防護現況

| 攻擊面 | 防護方式 | 狀態 |
|---|---|---|
| 用 APK 內的 anon key 列舉／下載全部使用者的發票 | Storage RLS 限 `authenticated` + 僅限自己資料夾 | ✅ |
| 惡意刪除或覆寫他人上傳的圖片 | 同上 | ✅ |
| 拿到圖片網址直接瀏覽發票 | bucket 設為私有 + 顯示時才發簽名網址（1 小時效期） | ✅ |
| 客戶端竄改 `is_premium` 白嫖永久版 | RLS 禁止客戶端寫入，只能由 `verify-purchase` 以 service role 寫 | ✅ |
| 退款後仍保有永久版 | 驗證時檢查 `purchaseState`，非 0 一律拒絕 | ✅ |
| 退款不撤銷造成的死收據死結 | 還原時偵測 `not_purchased` 自動 consume，釋放後可重新購買 | ✅ |
| 刪除他人帳號 | `delete-account` 從 JWT 解出使用者，不採信 App 傳入的 user_id | ✅ |

## 目前進度

最後更新：2026-08-05

### 一句話總結

**2.0.0 (34) 已於 2026-08-01 通過審核並發布於 Google Play 正式版** 🎉
（提交 ID 30，21:48 提交後快速通過）；死收據 consume 機制、
全手機送審管線、機構帳戶轉換（D-U-N-S 申請）皆於同日完成或啟動。

### 版本佈局

| 軌道 | 版本 | 說明 |
|---|---|---|
| **正式版（已發布）** | 34 (2.0.0) | 2026-08-01 通過審核上線 |
| 封閉測試 Alpha | 33 (2.0.0) | 與 34 同批打包，實機完整驗證用（含 consume 全流程考題） |
| release 管線 | 32 (2.0.0) | 同內容之管線產物，本次未用於送審；未來一律用此管線 |

### 已驗證的關鍵流程

| 流程 | 驗證結果 |
|---|---|
| 購買永久版 | 付款 → `verify-purchase` 驗證 → 寫入雲端 → 解鎖 ✅ |
| 登出後重新登入 | 永久版狀態保留 ✅ |
| 還原購買 | 正常還原 ✅ |
| 已退款的收據 | 正確拒絕解鎖（`purchaseState=1` → 400）✅ |
| **死收據完整閉環** | 退款(不撤銷) → 還原跳「訂單已失效」並 consume → 重買進付款畫面 → 付款成功重新解鎖 ✅ |
| **撤銷授權後全新購買** | 退款(勾移除授權) → 查無購買紀錄 → 可全新購買 ✅ |
| 刪除帳號 | Storage 圖片、課程、`user_status`、auth 帳號全數清除 ✅ |
| 刪除後重新登入 | 可用同一 Google Play 帳號還原永久版（文案已同步）✅ |
| 清除 App 資料後重登 | 課程與圖片完整還原 ✅ |
| 私有 bucket 圖片顯示 | 簽名網址正常，舊公開網址回 404 ✅ |

### 2026-08-01 完成事項

- [x] 業主指定文案更新：永久版感謝文案（含換行）、訂單失效對話框（含「您可隨時重新購買解鎖。」）
- [x] **死收據 consume 機制**寫入 `markPremium`（統一收尾：acknowledge / consume / 保住付款）
- [x] **production AAB 自動化管線**：新增 `.eas/workflows/build-android-production.yml`，
      建立 `release` 分支，手機即可完成「合併 PR → 打 AAB → 下載 → 上傳」全流程
- [x] 33 (2.0.0) 上封閉測試 Alpha，實機完整驗證 consume 閉環 — 全數通過
- [x] **34 (2.0.0) 送正式版審核 → 當晚通過並發布上線** 🎉
- [x] **Play 商店頁面抽驗通過**：已顯示版本 2.0.0、更新日期 2026/8/1、內購 $500、
      新黏土風圖示、下載大小 21.45 MB；「提供者」目前顯示 YANG YU HSIN（個人帳戶），
      機構帳戶轉換完成後將改顯示 Silent Love Studio
- [x] **鄧白氏 D-U-N-S 免費申請送件**（經 developer.apple.com/enroll/duns-lookup），
      啟動 Play Console 個人 → 機構帳戶轉換；英文名統一為 **Silent Love Studio**

### 2026-08-05 進行中 — Google Payments 地址變更（卡關，轉客服處理中）

- 目標：北平路個人住址 → **營業地址（400 台中市中區民權路164號20樓之11）**；
  舊地址仍為已驗證狀態，**收款與撥款不受影響，零急迫性**
- 歷程：公示資料頁截圖被退（不支援的文件類型）→ 改上傳**商業登記抄本** →
  送出 5 分鐘即遭**系統自動判讀退回**：「系統無法使用你提供的 ID 或文件驗證新地址」，
  地址變更卡在審核階段鎖定
- 推測原因：個人帳戶的機器驗證預期**個人名義**文件；抄本主體為工作室，
  人地連結在負責人欄位，OCR 讀不懂
- 已透過 Play 說明中心建立**支援請求 ID 8-0514000041008**（回信至 amanda 信箱），
  Google 商家小組（Daryl）已回覆：轉後端專員調查中，等待下一封回覆
- ⚠️ 應對原則：等待期間**勿再自行上傳任何文件**（驗證次數有上限）；
  文件只透過 **Google 安全表單**提交——任何要求「直接回信附上證件」的信件視同釣魚
- 備案：若民權路有**租賃契約**可作地址證明（接受清單第 4 項，親簽姓名＋地址機器好讀）
- 📌 Play Console 開發人員帳戶的「法定地址」仍**刻意不改**，等機構轉換一併處理

### 待處理

- [ ] 等 Google 商家小組回覆（支援請求 ID **8-0514000041008**，Daryl 已轉專員調查；
      回信寄至 amanda 信箱）——Payments 地址變更後續依專員指示辦理
- [ ] 等 Play 快速審查：隱私權政策網址＋資料安全表單兩筆變更（通常數小時內）
- [ ] 等鄧白氏編碼（Case Number **10741935**，Add business - Identity Data Only 免費軌道，
      已收 D&B 系統確認信）→ 完成 Play Console 機構帳戶轉換 → 開發者名稱改顯示
      **Silent Love Studio**、法定地址換營業登記地址；**勿購買付費認證**（業務推銷之
      三年期方案 NT$105,000 已拒絕，勿簽勿匯）；商業登記抄本已在手可作轉換佐證文件
- [ ] 低優先收尾：Google Cloud OAuth 同意畫面支援信箱（需先把工作室帳號加入 GCP 專案）、
      LINE Channel 的 Email
- [ ] personaltest282 有空時以無痕視窗健檢能否登入（未來送審備用）；
      不行則開新帳號並更新「應用程式存取權」（更新帳密不需重新送審）
- [ ] iOS TestFlight 實機驗證（等 iPhone）

### 2026-08-05 完成事項（二）— 隱私權政策頁搬遷自有網域

- [x] 確認網域註冊商為 **Cloudflare**（2026-06-26 註冊，DNS 亦由 Cloudflare 管理）
- [x] Cloudflare 新增 CNAME：`privacy` → `amanda1413191111-png.github.io`
      （**Proxy 必須 DNS only 灰雲**，橘雲會擋 GitHub 憑證簽發）
- [x] GitHub Pages 綁定 custom domain `privacy.silentlovestudio.com`，
      DNS check 通過＋Enforce HTTPS 已啟用
- [x] 新網址驗證正常；**舊 github.io 網址自動轉址至新網域，零斷鏈**
- [x] Play Console 送審兩筆變更：隱私權政策網址改新網域、
      資料安全表單兩處刪除網址（刪除帳戶／刪除資料）改新網域
- [x] delete-account.html 第五節連回隱私權政策的內文連結改為**相對路徑 `index.html`**
      （已上傳並以 commit e95e3f6 驗證，全站零 github.io 殘留）
- ⚠️ repo 內 GitHub 自動建立的 **CNAME 檔案不可刪除**（刪除會使自訂網域綁定失效）

### 2026-08-05 完成事項 — 對外聯絡資訊統一

- [x] 隱私權政策 `index.html` 與 `delete-account.html` 信箱改為
      **silentlovestudio.115@gmail.com**（已上傳 GitHub，線上頁面驗證生效）
- [x] Play Console 商店資訊聯絡詳細資料：支援 Email 改工作室信箱、
      網站改 **https://silentlovestudio.com/**
- [x] 開發人員帳戶「開發人員設定檔」公開 Email 改工作室信箱，**已通過驗證**
      （不需等機構轉換）
- [x] Google Payments 商家聯絡資訊（早已統一為工作室網域＋信箱）
- [x] App 程式碼掃描確認無 amanda 信箱（意見回報走 Supabase，免改免 build）
- [x] Play 商店頁面**實際確認生效**：「應用程式支援」與「開發人員資訊」兩處
      Email 均已顯示 silentlovestudio.115@gmail.com（快取數小時內刷新）

---

## 相關連結

- 官方網站：https://silentlovestudio.com/
- 隱私權政策：https://privacy.silentlovestudio.com/
- 刪除帳號說明：https://privacy.silentlovestudio.com/delete-account.html
- （舊網址 https://amanda1413191111-png.github.io/course-shelf-privacy/ 自動轉址至新網域）
- 對外聯絡信箱：silentlovestudio.115@gmail.com

---

© 2026 無聲有愛工作室. All rights reserved. 詳見 [LICENSE](./LICENSE)。
