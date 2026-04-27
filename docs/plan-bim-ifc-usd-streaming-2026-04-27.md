# 路線 A：離線 IFC → USD + Kit Streaming Server 執行手冊

日期：2026-04-27  
狀態：Active  
適用 repo：`C:\Repos\active\iot\AI-BIM-governance\bim-streaming-server`

## 結論

- 本 repo 的 streaming server **不直接吃 `.ifc`**：`omni.usd.UsdContext.open_stage_async` 只接受 USD 家族（`.usd` / `.usda` / `.usdc` / `.usdz`），且 build cache 內未包含任何 IFC / CAD importer extension。
- 採行 NVIDIA 官方推薦做法：**離線把 IFC 轉成 USD，再讓 Kit streaming server 載入 USD**。
- 工具選擇：**USD Composer + CAD Importer**。
- 模型放置：**repo 內 `./bim-models/`**（不入版控，由 `.gitignore` 規則阻擋）。
- Server 透過 `--/app/auto_load_usd=<usd 絕對路徑>` 啟動載入；client 端可透過既有 `openStageRequest` event 動態切換 stage，**`source/` 程式碼零修改**。

## 1. 前置安裝

- [Omniverse Launcher](https://www.nvidia.com/en-us/omniverse/) → 安裝 USD Composer
- 啟用 CAD Importer extension：USD Composer → `Window` → `Extensions` → 搜尋 `cad` 或 `ifc`，啟用 `omni.kit.cad_converter` / IFC importer 模組
- GPU / 驅動：比照本 repo 的 [README §環境需求](../README.md#環境需求)
- 確認本 repo 已可 build：

```powershell
cd C:\Repos\active\iot\AI-BIM-governance\bim-streaming-server
.\repo.bat build
```

## 2. IFC → USD 轉檔（USD Composer GUI）

1. 開啟 USD Composer。
2. `File` → `Import` → 選 `.ifc`。
3. CAD Importer 選項面板建議設定：
   - 保留 hierarchy（保留 IFC 的 spatial structure，便於 `getChildrenRequest` / `selectPrimsRequest` 使用）
   - 單位設 metric（公尺）；`metersPerUnit` 對齊
   - Material 採 USDPreviewSurface
   - 視需要關閉 / 簡化 mesh decimation；初次建議不要過度 decimate
4. 匯入完成後，`File` → `Save As`：
   - 路徑：`C:\Repos\active\iot\AI-BIM-governance\bim-streaming-server\bim-models\<案件名稱>.usd`
   - 大型模型建議改存 `.usdc`（binary，載入更快）
5. 命名建議：`<project>-<level>-<yyyymmdd>.usd`，例如 `ezplus-hq-l03-20260427.usdc`。
6. 確認 Y-up：USD Composer 預設 Y-up，與 Kit `omni.usd_viewer` template 一致；若 IFC 來源是 Z-up，匯入時可在 CAD Importer 軸向設定校正。

## 3. 模型放置與版控

- 路徑：`./bim-models/`（repo root 之下）
- `.gitignore` 已加入：
  - `/bim-models/*` 排除整個資料夾
  - 例外保留 `.gitkeep` 與 `README.md` 確保資料夾入版控
  - 全域 `*.usd` / `*.usda` / `*.usdc` / `*.usdz` 排除，避免任何位置誤 commit USD
  - `/source/apps/**` / `/source/extensions/**` / `/templates/**` 內的 USD 例外保留，不擋 NVIDIA template 範例
- 跨機 / 團隊共享請走 Nucleus 或外部物件儲存，**不要把 USD commit 到本 repo**。

驗證 ignore 規則：

```powershell
cd C:\Repos\active\iot\AI-BIM-governance\bim-streaming-server
git check-ignore -v bim-models/dummy.usd
git check-ignore -v bim-models/.gitkeep
git check-ignore -v source/apps/whatever.usd
```

預期：

- `bim-models/dummy.usd` 命中 `/bim-models/*`
- `bim-models/.gitkeep` 不被忽略（命中 `!/bim-models/.gitkeep`）
- `source/apps/whatever.usd` 不被忽略（命中 `!/source/apps/**/*.usd`）

## 4. 啟動 streaming server 並自動載入

### 4.1 推薦指令（headless + 絕對路徑）

```powershell
cd C:\Repos\active\iot\AI-BIM-governance\bim-streaming-server
.\repo.bat launch -n ezplus.bim_review_stream_streaming.kit -- --no-window --/app/auto_load_usd=C:/Repos/active/iot/AI-BIM-governance/bim-streaming-server/bim-models/<file>.usd
```

注意事項：

- 路徑用 forward slash（`/`），避免 Windows cmd 反斜線轉義問題。
- 一律加 `--no-window`：headless 路徑可避免 `FrameGrabFailed` / 視窗高度漂移，理由詳見 [`docs/issue-webrtc-framegrabfailed-resolution-mismatch-2026-04-27.md`](./issue-webrtc-framegrabfailed-resolution-mismatch-2026-04-27.md)。
- `auto_load_usd` 走的是 `setup.py` 的 `__open_stage`（`source/extensions/ezplus.bim_review_stream.setup/ezplus/bim_review_stream/setup/setup.py:49-70` / `:103-129`），用 `carb.tokens.get_tokens_interface().resolve()` 後再 fallback `Path.exists()`，**絕對路徑最穩**；相對路徑 `./bim-models/...` 在這條路徑沒有像 `openStageRequest` 一樣特別處理 `${app}/..`，請避免於 `auto_load_usd`。
- `auto_load_usd` 即使 `content.emptyStageOnStart=true` 也會生效；該 flag 只在 `setup.py:134` 控制是否還原 render-settings。

### 4.2 啟動成功 log

```text
Started primary stream server on signal port 49100 and stream port 47998
app ready
Sending message to client that stage has loaded: <url>
```

若沒看到 stage loaded log，加 `--info` 或 `--/app/printConfig=true` 重跑：

```powershell
.\repo.bat launch -n ezplus.bim_review_stream_streaming.kit -- --no-window --/app/auto_load_usd=C:/Repos/active/iot/AI-BIM-governance/bim-streaming-server/bim-models/<file>.usd --/app/printConfig=true --info
```

## 5. Web client 連線

對應 client repo：`C:\Repos\active\iot\AI-BIM-governance\web-viewer-sample`

```powershell
cd C:\Repos\active\iot\AI-BIM-governance\web-viewer-sample
npm run dev -- --host 127.0.0.1
```

`stream.config.json` 維持：

```json
{
  "source": "local",
  "local": { "server": "127.0.0.1", "signalingPort": 49100, "mediaPort": null }
}
```

開啟瀏覽器 `http://127.0.0.1:5173/`，選 `UI for default streaming USD Viewer app`，按 `Next`。

預期：

- Server log：`Client connected to WebRTC server`
- Browser：`video.readyState=4`、`videoWidth/Height>0`、`currentTime` 持續推進、無 `FrameGrabFailed` / `NoVideoPacketsReceivedEver`
- **Client 端不要寫死 `width` / `height`**；`omni.kit.livestream.webrtc` 會透過 `streamInfo` 自動協商

## 6. 動態切換模型（不重啟 server）

Server 已具備 `openStageRequest` event handler：`source/extensions/ezplus.bim_review_stream.messaging/ezplus/bim_review_stream/messaging/stage_loading.py:120-186`。

Client 透過 livestream messaging API 送：

```json
{
  "event_type": "openStageRequest",
  "payload": { "url": "<url>" }
}
```

`url` 接受三種寫法（由 `process_url` 在 `stage_loading.py:138-153` 處理）：

| 寫法 | 解析後實際讀取的位置 |
|---|---|
| `./bim-models/<file>.usd` | `${app}/../bim-models/<file>.usd`，即 `_build/windows-x86_64/release/bim-models/<file>.usd`（**注意：build 產物目錄，非 repo root**） |
| `./samples/<file>.usd` | `${omni.usd_viewer.samples}/samples_data/<file>.usd`（內建範例專用） |
| `C:/Repos/active/iot/AI-BIM-governance/bim-streaming-server/bim-models/<file>.usd` | 絕對路徑直接解析 |
| `omniverse://<server>/<path>/<file>.usd` | 透過 `carb.tokens` resolve 後交 `omni.client` 載入 |

**結論**：`./bim-models/...` 的 `${app}/..` 推導其實落在 `_build` 目錄而非 repo root，與 `bim-models/` 實際位置不一致。若要用 relative，**請在 build 後把 USD 放到 `_build/windows-x86_64/release/bim-models/`**；最穩的做法仍是傳**絕對路徑**或 Nucleus URL。

Server 收到後會回送：

- `openedStageResult`：`{ "url": "...", "result": "success" | "error", "error": "..." }`
- 載入過程中的 `updateProgressAmount` / `updateProgressActivity`

## 7. 驗證 checklist

啟動後逐項確認：

- [ ] USD Composer 匯入 `.ifc` 無錯誤；輸出 `.usd` / `.usdc` 可在 USD Composer 獨立開啟並看到正確幾何
- [ ] `git status` 在 `bim-models/` 內**不**顯示新放入的 USD（gitignore 生效）
- [ ] `git check-ignore -v bim-models/<file>.usd` 命中 `/bim-models/*`
- [ ] `git check-ignore -v bim-models/.gitkeep` 命中 `!/bim-models/.gitkeep`
- [ ] Streaming server log 出現 `Started primary stream server on signal port 49100 and stream port 47998` 與 `app ready`
- [ ] Stage 開啟 log 出現 `Sending message to client that stage has loaded: <url>`
- [ ] Browser 視訊：`video.readyState=4`、`videoWidth>0`、`videoHeight>0`、`currentTime` 推進
- [ ] Console 無 `FrameGrabFailed` / `NoVideoPacketsReceivedEver` / `Cannot stream video frame with resolution`
- [ ] 透過 `openStageRequest` 切換到第二份 `.usd` 成功（server log 應出現 `Received message to load`）

## 8. Troubleshooting

| 現象 | 處理 |
|---|---|
| 黑畫面 / `FrameGrabFailed` | 確認啟動有加 `--no-window`；client 不寫死高度。詳見 [issue 文件](./issue-webrtc-framegrabfailed-resolution-mismatch-2026-04-27.md) |
| `auto_load_usd` 沒生效 | 加 `--/app/printConfig=true --info` 重跑，確認 setting 真的被解析；改成絕對路徑 |
| IFC 太大 / 載入慢 | 另存 `.usdc`（binary）；於 USD Composer 評估 mesh decimation；關閉 material distill |
| IFC 單位不對 | USD Composer Import 選項中校正 `metersPerUnit`；或在轉檔後於 USD 加 `metersPerUnit` metadata |
| 模型出現但材質黑掉 | 檢查 IFC material 是否成功映射為 USDPreviewSurface；必要時手動補 MDL |
| `Failed to start the primary stream server` | 先檢查 Windows `Hardware-accelerated GPU scheduling` 設定（同 issue 文件 §Windows GPU Scheduling 前置條件） |
| Stage 切換失敗 | 看 server log 中 `openedStageResult` 的 `error` 欄位；常見是 URL 解析後檔案不存在 |

## 9. 後續可擴充（不在本次範圍）

- **路線 B**：把 `omni.kit.cad_converter` 加進 `ezplus.bim_review_stream.kit` 的 `[dependencies]`，讓 server 端直接吃 `.ifc`。需驗 Kit 110 相容性與 NVIDIA registry 取用方式。
- **路線 C**：另起一個 IFC pipeline 服務（ifcopenshell / 自製 USD writer），server 維持 USD-only。適合多人協作或 CI 自動化情境。
- **Nucleus 整合**：模型統一放 Nucleus，URL 改 `omniverse://<server>/<path>/<file>.usd`，team 端無需各自下載 USD。

## 10. 官方參考

- [NVIDIA-Omniverse/kit-app-template](https://github.com/NVIDIA-Omniverse/kit-app-template)
- [Application Streaming](https://docs.omniverse.nvidia.com/kit/docs/kit-app-template/latest/docs/streaming.html)
- [kit-app-template Tooling Guide](https://github.com/NVIDIA-Omniverse/kit-app-template/blob/main/readme-assets/additional-docs/kit_app_template_tooling_guide.md)
- [Kit Kernel Command Line Options](https://docs.omniverse.nvidia.com/kit/docs/carbonite/latest/docs/Kernel/CommandLineOptions.html)
- [Kit Manual: Configuration](https://docs.omniverse.nvidia.com/kit/docs/kit-manual/105.1/guide/configuring.html)
- USD Composer / CAD Importer：請參考 Omniverse Launcher 內附文件與 NVIDIA Omniverse 官網最新版本說明

## 相關文件

- [README.md](../README.md)
- [BUILD.md](../BUILD.md)
- [issue-webrtc-framegrabfailed-resolution-mismatch-2026-04-27.md](./issue-webrtc-framegrabfailed-resolution-mismatch-2026-04-27.md)
- [todo-webrtc-server-reboot-checklist-2026-04-24.md](./todo-webrtc-server-reboot-checklist-2026-04-24.md)
