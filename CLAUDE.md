# CLAUDE.md — RT-Handover

> **這份是給下次 Claude 看的工作上下文，不是文件。**
> 判斷標準只有一個：下次 Claude 讀完，能不能直接動手？
> 維護章法見 `SELA-Starter-Kit/conventions/CLAUDE-MD-章法.md`，每次升版前複習。

---

## 〇、當前狀態

- **版本：** V0.3.0
- **狀態：** 可運作、可部署 GitHub Pages
- **一句話定位：** 放腫科交班單——線上空白範本，瀏覽器內填寫，另存唯讀鎖定版給代理醫師
- **技術棧：** 純 HTML + CSS + Vanilla JS（單檔 `index.html`）
- **入口點：** `index.html`（本機開啟或 GitHub Pages）
- **UI 顯示名稱：** 彰濱秀傳放射腫瘤科 病人交班單（zip／資料夾名依 SPEC §10.5 用英文 `RT-Handover`）

---

## 一、技術棧決策（為什麼這樣選）

| 選擇 | 替代品 | 選這個的理由 |
|------|--------|------------|
| 純靜態單頁 + GitHub Pages branch-serve | Railway 後端 | V0.3.0 起頁面為空白範本、零個資，符合公開部署條件；零成本零維運；**不可加 build step**（branch-serve，見 Kit reference-static-pages §7） |
| **零個資架構**：範本公開、填寫在瀏覽器、鎖定版落地本機 | 後端儲存病人資料 | 個資完全不經伺服器、不進 repo；`.gitignore` 擋 `交班單_鎖定版_*.html` |
| **鎖定版＝無 script 靜態快照**（clone → 轉純文字 → 拔互動） | 序列化可再編輯版（V0.1–V0.2 做法） | SELA 要求不可改；無 script 也一併消滅重開重建問題（原坑 P1 整個失效，架構性解法優於 guard） |
| contenteditable（文字欄） | `<textarea>` | 列印視覺自然、內容直接在 DOM |
| `<input type=number>`（三數值欄） | contenteditable | 數值驗證與連動計算 |
| `<select>`（癌別、副作用） | datalist | 固定清單受控；副作用另配補充手寫欄（`.se-extra`） |
| 北歐橄欖綠 `#71805A` | 北歐霧藍 / 深橄欖 #48541D | SELA 兩階段指定：橄欖色相＋北歐霧面低飽和質感；SELA logo 微標維持橘白 |

---

## 二、業務對映表

（V0.3.0 暫無 — 單檔小工具。結構穩定後若出現「同概念散在 3+ 檔」再立表）

## 三、關鍵檔案路徑

| 想改什麼 | 動哪 |
|---------|------|
| 癌別／副作用選項 | script 開頭 `CANCER_TYPES`、`SIDE_EFFECTS` 陣列 |
| 連動計算（remain = total − done；cum = dose × done） | `recalc(card)` |
| 「即將完成」門檻（≤ 5 次） | `recalc()` 內 `remain <= 5` 兩處 |
| 卡片結構（欄位增減） | `makeCard()` 的 template literal |
| 空狀態提示 | `#empty-hint`，顯示邏輯在 `renumber()` |
| **鎖定版轉換規則** | `saveLocked()`：attribute 同步 → clone → contenteditable/input/select 轉純文字 → 拔 `.add-row/.del-btn/#saveBtn/#empty-hint/script` → 加 `lock-tag` → 移除 favicon link 與微標 img（單檔自包含） |
| 配色 | `:root` CSS 變數（主色 `--primary:#71805A`） |
| 版本號 | `VERSION` 常數（UI 兩處自動帶入） |

---

## 四、踩過的坑（編號累積，永不重排）

- **P1｜〔deprecated V0.3.0〕另存版重開後 script 重建洗掉內容**
  V0.1–V0.2 用 `body data-saved` guard 解。V0.3.0 起鎖定版**完全不含 script**，問題架構性消失。編號保留供追溯：若未來恢復「可再編輯的另存版」，此坑立即復活，guard 寫法見 V0.2.1 的 index.html（git 歷史）。

- **P2｜`<input>` 的即時值不會被序列化／clone**
  症狀：另存檔數字回到初始值。
  原因：輸入只改 DOM property，`cloneNode` 與 `outerHTML` 都只帶 attribute。
  做法：`saveLocked()` 第一步 `el.setAttribute("value", el.value)`。**clone 前必做**，新增 input 欄位都要納入。

- **P3｜`<select>` 的選擇同樣不進序列化／clone**
  同 P2。change 委派即時 `option.toggleAttribute("selected", ...)` 寫回，`saveLocked()` 開頭再保險同步。V0.3.0 起委派改判 `e.target.tagName === "SELECT"`，新加的下拉（副作用）自動涵蓋，不用逐一註冊。

- **P4｜鎖定版單檔寄送時外部資源全部斷鏈（V0.3.0）**
  症狀：鎖定版 email 給代理醫師，favicon 與 SELA 微標破圖。
  原因：`favicon/` 是相對路徑，單檔離開資料夾就找不到。
  做法：`saveLocked()` 直接移除 favicon `<link>` 與微標 `<img>`（保留文字 credit）。原則：**鎖定版必須自包含**——未來加任何外部資源（圖、字型、CDN）都要在此函式處理（內嵌或移除）。

- **#3（Kit）｜資料 vs 顯示分離**
  三個數值 input 是單一真相，remain/cum 純推導顯示，不回寫、不 parse 顯示文字。

- **#26／#27（Kit 種子）｜動態 HTML 與事件冒泡**
  卡片 innerHTML 建構＋事件委派（input/change/click），新增卡片自動享有互動。

- **#14（Kit）｜版本號雙檔同步**
  三處：zip 檔名、`VERSION` 常數、README／CLAUDE.md。每次升版全查。

---

## 五、煙霧測試

```
1. 開 index.html → 0 張卡片、空狀態提示可見、標題旁 V0.3.0
2. ＋新增病人 → 空狀態提示消失、卡片編號 01、徽章「資料未完整」
3. 填三個數值 → 剩餘與累計即時算；≤5 轉「即將完成」
4. 癌別選一項、副作用選一項＋補充欄手寫 → 都保留
5. 🔒 另存鎖定版 → 開下載檔驗證：
   - 無 <script>、無 input/select/contenteditable（全轉純文字）
   - 無新增／刪除／另存按鈕，僅剩「列印」
   - 標題旁有「🔒 鎖定版・唯讀」標記
   - 填寫內容完整（含副作用補充欄）、無破圖
6. 刪除卡片（confirm）→ 編號重排；刪到 0 張 → 空狀態提示回來
7. Ctrl/Cmd+P → 工具列、新增鈕、刪除鈕、空狀態提示全隱藏
8. 部署驗證：推 main → Pages 開啟 → 頁面原始碼無任何病人資料
```

---

## 六、版本歷程

- **V0.3.0（2026-07）** 轉為 GitHub Pages 可部署的零個資架構：預設病人清空（空白範本＋空狀態提示，個資不進 repo）；副作用改下拉（16 項）＋補充手寫欄；另存改產生**無 script 唯讀鎖定版**（P1 deprecated、新增 P4 自包含原則）；`.gitignore` 擋鎖定版檔名。
- **V0.2.1（2026-07）** 配色改北歐橄欖綠 `#71805A`；移除頁尾簽章列。
- **V0.2.0（2026-07）** 主色橄欖綠；已完成次數可編輯、剩餘自動推導；癌別下拉（P3）；新增／刪除病人；產出 SELA-handoff.md。
- **V0.1.0（2026-07）** 初版。12 位病人卡片、數值連動、列印、可再編輯另存版（P1/P2）、北歐霧藍。

---

## 七、下版候選工作

1. **鎖定版內嵌 SELA 微標（SVG inline 或 data-URI）** — P4 目前用「移除」解破圖，但鎖定版是交到代理醫師手上的門面，品牌歸屬印記不該缺席；把 sela.svg 內容 inline 進鎖定版即可，成本低且符合雙軌微標鐵律
2. 交班期間自動帶入民國日期預設值
3. 病人排序切換（依剩餘次數升冪）
4. App logo（雙軌系統 §8；待 SELA 委託生成後整合）

---

## 八、升版必讀

### ⚠ V0.3.0 架構轉向：零個資 + 唯讀鎖定版

- **不要**把任何病人資料寫回 `index.html`（含範例、測試資料）——頁面已公開部署
- **不要**恢復「可再編輯的另存版」除非 SELA 明說；若恢復，P1 guard 要一起回來
- 動 `saveLocked()` 前先讀坑 P2/P3/P4——attribute 同步在 clone 前、自包含原則

---

> **一句話總結：** 公開空白範本＋瀏覽器填寫＋無 script 鎖定版三段式；`saveLocked()` 的順序（attribute 同步 → clone → 轉純文字 → 拔互動 → 自包含檢查）是本專案最脆的一段，動之前讀第四章 P2/P3/P4。
