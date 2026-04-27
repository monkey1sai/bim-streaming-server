# Issue Wiki: WebRTC `FrameGrabFailed` 排查與解法

日期：2026-04-27

## Issue 摘要

`ezplus.bim_review_stream_streaming.kit` 在 `client -> WebRTC -> server` 流程中，瀏覽器端會出現 `Stream disconnected from server, FrameGrabFailed.`，導致 `localhost:5173` 雖然能連到 `127.0.0.1:49100`，但畫面無法穩定顯示。

這次的最終根因不是 stage 是否為空，也不是 signaling port 設錯，而是 `client` 和 windowed `Kit` server 在串流建立時使用了不同的解析度協商值；而且這個高度不是固定常數，會隨 Windows 實際可用視窗區域而漂移。

## 影響範圍

- `server` repo：`bim-streaming-server`
- `client` repo：`C:\Repos\active\iot\web-viewer-sample`
- 受影響流程：
  - `ezplus.bim_review_stream_streaming.kit`
  - `web-viewer-sample` local mode
  - `localhost:5173` -> `ws://127.0.0.1:49100/sign_in`

## 現象

### 瀏覽器端

- local client 可成功載入 `http://localhost:5173`
- 可成功連到 `127.0.0.1:49100`
- 曾經可看到 `video` element 存在，但最終會出現：

```text
Stream disconnected from server, FrameGrabFailed.
```

或在後續測試中出現：

```text
Streaming stopped as NoVideoPacketsReceivedEver.
```

### Server 端

`server` 在有視窗模式下，會重複輸出：

```text
Cannot stream video frame with resolution `1920x1062` that differs from that of 1920x1080 established when the client connected to the stream.
```

在更差的情況下，還會進一步出現：

```text
Device lost
Failed to begin render graph. Device lost detected while waiting for frame submission semaphore.
A GPU crash occurred. Exiting the application...
```

## 與空 template 的關係

這次要區分兩件事：

1. `content.emptyStageOnStart = true`
   - 這會造成 `UI for any streaming app` 進來後看到黑畫面。
   - 這只能解釋「空 stage」，不能解釋 `FrameGrabFailed`。
2. `FrameGrabFailed`
   - 這次真正的 root cause 是解析度協商不一致。
   - 就算 client 已經成功送出 `openStageRequest`，只要解析度 mismatch 還在，畫面仍可能失敗或直接把 renderer 打到 `Device lost`。

## Root Cause

### 1. Server 實際 stream content area 不是固定值

第一次成功建立 session 後，client 從 stream metadata 拿到：

```json
"streamInfo":[{"width":1920,"height":1062,"fps":60}]
```

但在後續重新驗證時，server 端又出現：

```text
Cannot stream video frame with resolution `1920x1009` that differs from that of 1920x1062 established when the client connected to the stream.
```

再往下追，重新協商後甚至會變成：

```text
Cannot stream video frame with resolution `1920x1009` that differs from that of 1920x1008 established when the client connected to the stream.
```

這代表 windowed `Kit` server 在這台 Windows 主機上的實際可串流內容區高度會變動，不是永遠固定在 `1062`。

### 2. Client 初始連線若硬編碼固定高度，遲早會再失配

`web-viewer-sample/src/AppStream.tsx` 的 local mode 連線設定原本是：

```ts
width: 1920,
height: 1080,
fps: 60,
```

因此 WebRTC 連線建立時，client 先用 `1920x1080` 協商；但 server 真正送出來的是 `1920x1062`，導致 server 側一直丟出 resolution mismatch warning。

把 client 改成 `1062` 只能修掉當下那一輪，但只要 host 視窗內容區再變，例如這次後續又漂到 `1009`，client 就會再次失配。

### 3. 這次還有一個容易忽略的奇偶數問題

後續驗證時，server log 顯示：

```text
Processing static resize of video stream with expected extents 1920x1079 that are invalid so they have been adjusted to 1920x1078
```

這表示 stream SDK / encoder 對高度有額外限制：

- Height 必須是偶數

因此若實際內容區高度落在奇數，例如 `1009`，就很容易出現：

- client 看到或協商到 `1008`
- server 實際又在送 `1009`
- 最後形成 `1009` vs `1008` 的 1-pixel mismatch

### 4. Mismatch 會連帶觸發 frame grab / renderer 異常

這個 mismatch 並不只是 warning：

- 輕則 client 沒收到有效影格，最後變成 `NoVideoPacketsReceivedEver`
- 重則 server 側 RTX renderer 進入 `Device lost / GPU crash`

也就是說，`FrameGrabFailed` 在這次案例裡是「解析度協商不一致」的外顯結果，而不是第一層根因。

## 解法

這次要分成兩層解法：

1. 不要再假設 windowed server 的內容區高度是固定常數
2. 啟動 server 時，盡量把主視窗高度控制到能落在合法偶數內容區

本次實際可重現的 workaround 是：

- 啟動 windowed server 時，明確傳入：

```powershell
.\repo.bat launch -n ezplus.bim_review_stream_streaming.kit -- --/app/window/height=1079 --/app/window/width=1920
```

server 會把 `1079` 自動調整成可編碼的 `1078`，而最後 client 實測拿到穩定的：

```text
1920x1008
```

搭配 local client 目前的協商高度調整，可讓串流恢復正常。

檔案：

```text
C:\Repos\active\iot\web-viewer-sample\src\AppStream.tsx
```

調整前：

```ts
width: 1920,
height: 1080,
fps: 60,
```

第一次調整後：

```ts
width: 1920,
height: 1062,
fps: 60,
```

後續再次驗證後，因 host 視窗內容區已漂移，不再適合把 `1062` 視為永久答案。當前這台機器重新驗證可用的 local 協商值已改為：

```ts
width: 1920,
height: 1009,
fps: 60,
```

注意：這個值仍屬 host-specific workaround，並不代表所有時間點或所有機器都固定是 `1009`。

## 驗證流程

### 1. 啟動 server

使用 windowed 模式啟動。若要降低奇數高度造成的 mismatch，優先使用：

```powershell
.\repo.bat launch -n ezplus.bim_review_stream_streaming.kit -- --/app/window/height=1079 --/app/window/width=1920
```

驗證點：

- `kit.exe` 存活
- `49100` 由新的 `kit.exe` listen
- `app ready` 出現在 server log

### 2. 啟動 client

在 `web-viewer-sample` repo 執行：

```powershell
npm run dev
```

開啟：

```text
http://localhost:5173
```

UI 選擇：

- `UI for default streaming USD Viewer app`

### 3. 成功條件

瀏覽器端驗證：

- `video.readyState = 4`
- `video.videoWidth = 1920`
- `video.videoHeight` 與 server 最終穩定輸出的偶數高度一致
- `currentTime` 持續增加
- 沒有再出現：
  - `FrameGrabFailed`
  - `NoVideoPacketsReceivedEver`

Server 端驗證：

- 不再出現：

```text
Cannot stream video frame with resolution `1920x1062` that differs from that of 1920x1080 established when the client connected to the stream.
```

### 4. 本次實測結果

第一次修正後，曾確認：

- WebRTC session 可建立
- `video.readyState = 4`
- `video.videoWidth = 1920`
- `video.videoHeight = 1062`
- `currentTime` 持續前進
- sample scene 可正常顯示

但後續重跑同流程後，發現 `1062` 並不是固定值。當 Windows 實際工作區改變時，windowed server 的內容區會漂到 `1009`，進一步造成 `1009` vs `1008` 的 mismatch。

在重新以 `--/app/window/height=1079 --/app/window/width=1920` 啟動 server，並重新驗證後，已再次確認：

- WebRTC session 可建立
- `video.readyState = 4`
- `video.videoWidth = 1920`
- `video.videoHeight = 1008`
- `currentTime` 持續前進
- sample scene 可正常顯示

因此目前判定：

- `FrameGrabFailed` 的根因仍是解析度協商不一致
- 而更精確地說，是 windowed server 的內容區高度會漂移，且奇數高度會再放大 mismatch 風險

## 操作注意事項

### 1. 這個解法針對 windowed server 路徑

本次驗證成功條件建立在 windowed `Kit` server。若改用 `--no-window`，行為可能不同，需獨立驗證。

### 2. `UI for any streaming app` 仍可能只看到空畫面

若 app 本身沒有主動送 stage loading 流程，`UI for any streaming app` 不會替你送 `openStageRequest`。這是空 stage 問題，不是 `FrameGrabFailed` 問題。

### 3. 解析度不能只看 `.kit` 設定值

即使：

- `renderer.resolution.width = 1920`
- `renderer.resolution.height = 1080`
- `window.width = 1920`
- `window.height = 1080`

實際 stream content area 仍可能因 OS window chrome、工作列、host working area 或視窗狀態變成 `1920x1062`、`1920x1009` 或其他值。排查時應以實際 `streamInfo` 與 `video.videoWidth/video.videoHeight` 為準，不要只看 kit config。

### 4. Windows working area 會直接影響這個問題

本次後續驗證時，主螢幕資訊為：

- Screen bounds: `1920x1080`
- Working area: `1920x1032`

這表示工作列等系統 UI 已吃掉一部分高度。當 server 用 windowed 模式跑滿螢幕可用區時，實際內容區高度就可能繼續被壓縮，進而讓串流內容高度落在非預期值。

### 5. 奇數高度是高風險訊號

若 server log 出現像 `1920x1009` 這種奇數高度，應優先懷疑：

- encoder / stream SDK 會把它修正成相鄰偶數
- client / server 兩側看到的高度可能差 1 pixel
- 這會直接導致「有連上但沒有畫面」

## 後續建議

1. 若希望避免之後再被 host 視窗尺寸影響，應優先實作：
   - client 端依 `streamInfo` 自動調整或二次 `resize`
   - 或啟用/評估 `dynamicResize`
2. 在沒有動態調整前，不要把 `1062`、`1009`、`1008` 當成永久常數；每次 host 環境不同都可能變。
3. 若要用 windowed 模式做現場排障，建議優先把 server 啟成容易落在偶數內容區的高度，例如：
   - `--/app/window/height=1079 --/app/window/width=1920`
4. 若之後要恢復 `1920x1080` 協商，需先確認 server 實際能穩定輸出 `1920x1080`，否則同樣會重現 mismatch。
5. 若再次看到 `Device lost`，先確認前面是否先出現 resolution mismatch，不要直接把問題歸到 GPU driver。

## 架構角色說明

這次排查中，最容易混淆的是 `bim-streaming-server`、`49100`、`web-viewer-sample` 與 `5173` 分別代表什麼。

### 1. `bim-streaming-server` 是 Kit app + WebRTC stream server

`bim-streaming-server` 這個 repo 產出的 `ezplus.bim_review_stream_streaming.kit`，同時扮演兩個角色：

- `Kit viewer`：
  - 本機會開出 Omniverse / Kit 原生視窗
  - 視窗裡的主要內容是 `Viewport`
  - 這就是 server 端實際 render 的畫面來源
- `WebRTC stream server`：
  - 將上面的 viewport 畫面編碼後對外提供 WebRTC signaling / media
  - 本次主要驗證的 signaling 入口是 `49100`

也就是說，`bim-streaming-server` 自己有 viewer，但那個 viewer 是 `Kit 原生視窗`，不是獨立的瀏覽器頁面。

### 2. `web-viewer-sample` 是 browser viewer client 範例

`web-viewer-sample` 不是 stream server，本質上是：

- 一個示範如何在瀏覽器中嵌入 NVIDIA WebRTC streaming library 的 sample client
- 一個示範如何建立 WebRTC 連線、接收畫面、送控制訊息的範例 viewer

它的 `5173` 只是 Vite dev server 提供出來的前端頁面埠，作用是：

- 把 sample viewer 網頁送到瀏覽器
- 讓瀏覽器中的 JavaScript 再去連真正的 stream server

所以：

- `5173` 不是 WebRTC stream server
- `49100` 才是這次主要的 WebRTC signaling / streaming 入口

### 3. 別台電腦若無法開 `192.168.10.105:5173`，不代表 stream server 沒開

本次另行確認到：

- `49100` 是綁在 `0.0.0.0:49100`
- 代表 `Kit` 這一側的 stream server 允許其他 IP 嘗試連入

但 `web-viewer-sample` 的 Vite dev server 若只綁 `localhost` 或 `::1`，那別台電腦打：

```text
http://192.168.10.105:5173
```

仍然會失敗。這代表：

- 失敗的是 sample viewer 頁面沒有對外提供
- 不代表 `49100` 那個 WebRTC server 本身一定沒對外開

### 4. `web-viewer-sample` 預設 local mode 也可能把連線指到錯的主機

若 `web-viewer-sample` 的 `stream.config.json` 仍是：

```json
"local": {
  "server": "127.0.0.1",
  "signalingPort": 49100
}
```

那麼即使別台電腦成功打開 sample viewer 頁面，瀏覽器裡的 client 仍會去連：

```text
127.0.0.1:49100
```

也就是「別台電腦自己的 localhost」，而不是 `192.168.10.105:49100`。

### 5. 正確理解應該是這樣

本次架構可用以下文字理解：

```text
bim-streaming-server
  = Kit 原生 viewer + WebRTC stream server
  -> 對外提供 49100 等 streaming 端點

web-viewer-sample
  = browser viewer client sample
  -> 透過 5173 提供前端頁面
  -> 頁面載入後再去連 49100
```

換句話說：

- `bim-streaming-server` 有自己的 viewer，但那是 server 端原生視窗
- `bim-streaming-server` 沒有內建獨立的 web viewer 頁面
- `web-viewer-sample` 是「連線觀看用的前端範例」
- 若要跨機觀看，重點不是只有 `5173` 可不可達，還要確認 sample viewer 最後實際連到的是不是正確的 server IP
