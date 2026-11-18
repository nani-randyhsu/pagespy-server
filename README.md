# pagespy-server

[PageSpy](https://github.com/HuolalaTech/page-spy-web) 遠端除錯 server，部署在 Render 免費方案。
網頁或 App WebView 載入 PageSpy SDK 後，電腦瀏覽器打開 server 網址，就能看到頁面的 Console、Network、Storage 等資訊。

## 目錄結構

```
pagespy-server/
├── package.json        # 相依官方 npm 套件 @huolala-tech/page-spy-api，以及 start 指令
├── package-lock.json   # 鎖定版本（含各平台的執行檔套件，Render 上會裝 linux-amd64）
├── render.yaml         # Render Blueprint 設定
├── .gitignore          # 排除 node_modules、config.json、data/、log/
└── README.md
```

server 本體是官方發佈的 Go 執行檔，前端除錯介面和 SDK 都已經打包在裡面，不需要 build。

| 指令 | 做什麼 |
| --- | --- |
| `npm start` | 依 `PORT`（預設 `6752`）產生 `config.json`，再執行 `page-spy-api` |

- page-spy-api 只從工作目錄的 `config.json` 讀 port，不讀 `PORT` 環境變數，所以每次啟動都依 `PORT` 重新產生 `config.json`
- 執行時會在工作目錄產生 `config.json`、`data/data.db`（SQLite）、`log/`，都已排除在 git 外
- 有設 `AUTH_PASSWORD` 時，server 會把密碼寫進 `config.json`，**`config.json` 不可 commit**

## 使用方式

### 方式一：Mac 本機執行

```bash
npm install
npm start                          # http://localhost:6752
AUTH_PASSWORD=xxx npm start        # 要測試登入密碼時
```

### 方式二：部署到 Render

用 Blueprint 部署：Render → New → Blueprint → 選這個 repo，會讀 `render.yaml` 自動建立服務。建立時會要求填 `AUTH_PASSWORD`。

| 設定 | 值 |
| --- | --- |
| 服務名稱 | `pagespy-server` |
| 方案 / 區域 | free / singapore |
| Build | `npm ci` |
| Start | `npm start` |
| 自動部署 | `autoDeployTrigger: commit`，push 到連結的分支就部署 |
| Health check | `/` |
| 環境變數 | `AUTH_PASSWORD`：除錯介面的登入密碼，建立 Blueprint 時在 Render 後台填，不進 git |
|  | `JWT_SECRET`：Render 自動產生，讓登入狀態在 server 重啟後仍有效 |

- 部署後網址是 `https://pagespy-server.onrender.com`（服務名稱被占用時 Render 會加後綴，以 Render 後台顯示的為準）
- 免費方案閒置 15 分鐘會休眠，第一次打開要等約 1 分鐘叫醒
- 沒有自動部署的話，到服務的 Settings 確認 repo 是透過 GitHub 帳號連線，再用 Deploys → Manual Deploy → Deploy latest commit 補部署

## 頁面端接入

server 會提供 SDK：`https://<server>/page-spy/index.min.js`。頁面載入 SDK 後建立 `PageSpy` 實例即可，App WebView 可以用下面這段注入：

```js
(function () {
  if (window.__pageSpyInjected) return;
  window.__pageSpyInjected = true;

  var host = 'pagespy-server.onrender.com';
  var script = document.createElement('script');
  script.src = 'https://' + host + '/page-spy/index.min.js';
  script.crossOrigin = 'anonymous';
  script.onload = function () {
    window.$pageSpy = new PageSpy({
      api: host,                          // 只填 host，不含 https://
      clientOrigin: 'https://' + host,    // 除錯介面的網址
      enableSSL: true,
      project: 'OneBook',                 // 除錯介面可依 project 篩選
      title: 'iOS - <書本 id>',           // 顯示在房間列表，用來分辨裝置
      autoRender: false,                  // 不在頁面左下角顯示 PageSpy 浮動按鈕
    });
  };
  document.head.appendChild(script);
})();
```

接入後，電腦瀏覽器打開 server 網址，輸入密碼，在房間列表點對應的 `title` 開始除錯。

## 升級

```bash
npm install @huolala-tech/page-spy-api@<版本> --save-exact
git commit -am "升級 page-spy-api 到 <版本>"
git push        # Render 自動重新部署
```

版本與更新內容看 [page-spy-web releases](https://github.com/HuolalaTech/page-spy-web/releases)。

## 限制

- **免費方案存不住資料**：Render 免費方案的硬碟是暫時的，重新部署、休眠、重啟都會清掉 `data/` 和 `log/`。即時除錯不受影響，但上傳到 server 的離線 log 回放會消失。要保留得改用付費方案加 persistent disk，或在 `config.json` 設定 S3 相容儲存（`storageConfig`）與 MySQL（`databaseConfig.mysqlUrl`）
- **看不到 SDK 載入前的事**：SDK 是頁面載入後才注入，在那之前的 console log 和請求不會出現

## ⚠️ 安全注意

- `AUTH_PASSWORD` 只保護除錯介面（房間列表、log 管理）。SDK 建立房間與連線用的 API 是公開的
- 除錯介面看得到頁面的 storage 和 cookie（含登入 token），只讓 dev / uat build 接入，只用測試帳號
- 正式版 App 不可帶這個網址
