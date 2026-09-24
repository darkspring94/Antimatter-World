# Antimatter World 改造進度紀錄

本文件記錄 2026-09-24 這次 PR 做了哪些事、還有哪些原名沒改、以及合併後 Spring 要手動做的 GitHub 設定。之後每次改造都可以延續這份紀錄。

## 這次改了什麼

### 1. 改名（只改玩家看得到的地方）

| 檔案 | 改動內容 |
| :-- | :-- |
| [public/index.html](public/index.html) | 瀏覽器分頁標題 `<title>` 與 `<meta name>` 從 "Antimatter Dimensions" 改成 "Antimatter World" |
| [README.md](README.md) | 標題改成 "Antimatter World"，並加上出處說明（見下方第 2 點） |
| [src/components/CreditsDisplay.vue](src/components/CreditsDisplay.vue) | 製作名單頁面頂端的大標題改成 "Antimatter World" |
| [src/components/modals/CreditsModal.vue](src/components/modals/CreditsModal.vue) | 製作名單彈窗標題改成 "Antimatter World" |
| [src/components/modals/InformationModal.vue](src/components/modals/InformationModal.vue) | 「About the game」內文開頭改成介紹 "Antimatter World"，同時保留原作者 Hevipelle 在 2016 年創作 Antimatter Dimensions 的史實敘述（沒有竄改歷史），並在段落最後加上出處說明（見下方第 2 點） |

**沒有改的地方**（照 Spring 指示，這次不動）：
- 程式內部的變數名、函式名、檔名、CSS class 名稱（例如 `.antimatter-dimensions`、程式檔案 `antimatter-dimension.js`）
- npm 套件名稱，包含 `@antimatter-dimensions/notations`
- `InformationModal.vue` 裡連到 Discord／Google Play／App Store／Steam／Reddit 的外部連結文字（例如 "Antimatter Dimensions on Steam"）——**保留不改**，因為這些連結實際上指向的是原版遊戲在這些平台的官方頁面，不是 Spring 的版本；改了標籤但沒改連結內容會誤導玩家

### 2. 註明出處

- `LICENSE` 檔案與原作者版權聲明（`Copyright (c) 2017 IvarK`）**完全沒有刪改**
- 在 [README.md](README.md) 與遊戲內「About the game」頁面（[InformationModal.vue](src/components/modals/InformationModal.vue)）都加上：
  > Antimatter World is based on Antimatter Dimensions by IvarK and contributors (MIT License).
  > https://github.com/IvarK/AntimatterDimensionsSourceCode

### 3. 切斷原作者的外部服務

- **Google Analytics（vue-gtag）**：[src/core/ui.js](src/core/ui.js) 原本有 `Vue.use(VueGtag, { config: { id: "UA-77268961-1" } })`，會把資料送到原作者的 Google Analytics 帳號。已移除該段程式碼與對應的 `import`，遊戲不會再送出任何分析追蹤資料。
  - `package.json` 裡的 `vue-gtag` 套件本身**先保留沒刪**（純粹是因為本機沒有 Node/npm 可以重新產生 `package-lock.json`，貿然改 `package.json` 但沒同步更新鎖定檔，`npm ci` 在雲端可能會因為兩者對不上而失敗）。這個套件現在已經沒有任何程式碼在用它，之後 Spring 找一台有裝 Node 的環境跑一次 `npm uninstall vue-gtag` 就能把它連依賴一起清掉，不影響遊戲功能。
- **Firebase 雲端存檔**：檢查過 [src/core/storage/cloud-saving.js](src/core/storage/cloud-saving.js) 與 [src/core/storage/firebase-config.js](src/core/storage/firebase-config.js) 後確認**不需要改程式碼**——`firebase-config.js` 裡的 `apiKey` 等欄位本來就都是 `null`（Spring 的版本沒有設定 `FIREBASE_CONFIG`），程式碼原本就設計成：`apiKey` 是 `null` 時完全不會呼叫 `initializeApp()`，所有雲端存讀方法（登入、存檔、讀檔…）一進入就會直接 `return`，安全跳過，不會噴錯也不會卡住遊戲。本機存檔功能不受影響、正常運作。
- **Steam 相關檔案**：`.env.steam-development`、`.env.steam-release`、`src/steam/` 目錄，這次**完全沒動**，照 Spring 指示保持原樣。

### 4. 設定自動發布到 GitHub Pages

- 保留並修改 [.github/workflows/deploy-master.yml](.github/workflows/deploy-master.yml)：
  - Node 版本從寫死的 `"14"`（太舊）改成 `"lts/*"`（讓 GitHub Actions 每次都自動抓當下最新的 LTS 版本，以後不用再手動更新版本號）
  - 新增 `pull_request` 觸發條件：每次開 PR 到 `master` 時，會自動跑一次 `npm ci` + `npm run build:master`（純建置驗證，**不會**發布），確保改動沒有把打包弄壞才能合併——這次 PR 本身就是用這個機制在雲端驗證過建置成功
  - 實際發布到 `gh-pages` 分支的步驟（`JamesIves/github-pages-deploy-action`）加了 `if: github.event_name != 'pull_request'`，只有真正 push 到 `master`（或手動觸發）才會發布，PR 檢查時不會誤發布
  - **為什麼用 `build:master` 而不是 `build:release`**：`build:release` 打包出來的版本會被原本的 `deploy-release.yml` 推到**外部的 `IvarK/AntimatterDimensions` repo**（原作者的正式站），Spring 沒有那個 repo 的權限也不該推過去；`build:master` 才是打包成部署到「這個 repo 自己的 `gh-pages` 分支」的版本，符合 Spring 「push 到 master 就發布到自己的 GitHub Pages」的需求
- **刪除 `.github/workflows/deploy-release.yml`**：這個 workflow 的用途是把打包結果推送到原作者的官方 repo `IvarK/AntimatterDimensions`，Spring 沒有也不需要那個 repo 的存取權（`GH_PUBLISH_TOKEN` secret 也不存在），留著也不會動作，直接刪除避免混淆
- **刪除 `.github/workflows/stale.yml`**：這是原作者用來自動關閉沒人理的 issue／PR 的管理工具，Spring 的 fork 用不到，已刪除
- 已在雲端（GitHub Actions）實際跑過 `npm ci` 與 `npm run build:master`，確認沒有錯誤（結果請見這個 PR 的 Checks 頁籤）

## 還出現「Antimatter Dimensions」原名、這次沒改的位置

以下檔案裡還有 "Antimatter Dimensions" 字樣，這次先不動，理由分兩種：

**(A) 遊戲內文字（劇情、成就、教學、新聞、更新日誌等）**——照 Spring 指示，這次先不改，之後要改再另外處理：
- `src/core/secret-formula/achievements/normal-achievements.js`（成就文字）
- `src/core/secret-formula/changelog.js`（更新日誌）
- `src/core/secret-formula/news.js`（遊戲內新聞跑馬燈）
- `src/core/secret-formula/h2p.js`（新手教學 "How to Play"）
- `src/components/tabs/automator/AutomatorDocsIntroPage.vue`（Automator 功能說明文件）
- `src/core/secret-formula/challenges/*.js`、`src/core/secret-formula/eternity/*.js`、`src/core/secret-formula/infinity/*.js`、`src/core/secret-formula/reality/*.js`、`src/core/secret-formula/celestials/*.js`、`src/core/secret-formula/catchup-resources.js`（各種挑戰、天神、升華效果的文字說明）

**(B) 其實是遊戲機制的名稱，不是品牌名**——"Antimatter Dimensions" 在這些檔案裡指的是遊戲裡「第一到第八維度」這個核心機制的正式名稱（跟品牌名同名純屬巧合），改了會動到遊戲功能命名，不在這次「只改玩家看得到的品牌名」範圍內：
- `src/core/dimensions/antimatter-dimension.js`
- `src/components/tabs/antimatter-dimensions/ClassicAntimatterDimensionRow.vue`
- `src/components/tabs/antimatter-dimensions/ModernAntimatterDimensionRow.vue`
- `src/components/tabs/antimatter-dimensions/TickspeedRow.vue`
- `src/components/tabs/infinity-dimensions/ClassicInfinityDimensionsTab.vue`
- `src/components/tabs/infinity-dimensions/ModernInfinityDimensionsTab.vue`
- `src/components/tabs/statistics/MultiplierBreakdownEntry.vue`
- `src/components/tabs/statistics/MultiplierBreakdownTab.vue`
- `src/components/ui-modes/HeaderChallengeEffects.vue`
- `src/components/modals/SacrificeModal.vue`、`src/core/sacrifice.js`
- `src/components/modals/prestige/AntimatterGalaxyModal.vue`、`src/components/modals/prestige/DimensionBoostModal.vue`
- `src/components/tabs/options-saving/OptionsSavingTab.vue`
- `src/core/secret-formula/tabs.js`
- `src/game.js`、`src/utility/deepmerge.js`

## 合併 PR 之後，Spring 要在 GitHub 網頁上手動做的設定

1. 打開 repo `darkspring94/Antimatter-World` → 上方分頁點 **Actions**。如果看到「Workflows aren't being run on this forked repository」之類的提示，點 **"I understand my workflows, go ahead and enable them"** 按鈕啟用 Actions（Fork 的 repo 預設會關閉 Actions，這是 GitHub 的安全機制，需要手動打開一次）
2. 合併這個 PR 到 `master` 後，等個 1-2 分鐘，回到 **Actions** 分頁確認 **"Deploy master"** 這個 workflow 有跑完並顯示綠色勾勾（它會自動建立一個 `gh-pages` 分支）
3. 到 repo 上方 **Settings** → 左側選單 **Pages**
4. 在 **Build and deployment** 區塊，**Source** 下拉選單選 **"Deploy from a branch"**
5. **Branch** 下拉選單選 **"gh-pages"**，資料夾維持 **"/ (root)"**，按 **Save**
6. 等 1-2 分鐘，頁面上方會出現網址（類似 `https://darkspring94.github.io/Antimatter-World/`），點開確認遊戲能正常開啟、存檔功能正常（用瀏覽器 localStorage 本機存檔）

## 下一步建議

- 上面列的「還沒改的原名位置」如果之後想繼續改，建議先處理 (A) 遊戲文字類（成就、教學、新聞），這些是玩家最常看到的，且不涉及程式邏輯，風險低
- `InformationModal.vue` 裡「GitHub repository」按鈕目前還連到原作者的 repo，可以考慮之後改成連到 Spring 自己的 `darkspring94/Antimatter-World`，讓玩家能看到目前這個版本真正的原始碼
- 找時間在有裝 Node.js 的電腦上跑 `npm uninstall vue-gtag`，把不再使用的分析套件依賴清掉
- 之後如果想把遊戲圖示、favicon（`public/icon.png`）換成自己的美術風格，也可以列入下次改造清單
