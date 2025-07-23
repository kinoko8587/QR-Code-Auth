# QR Code Authentication System
QR Code authentication system, supporting millions of users to quickly clock in and out and authenticate their identity

## 🧱 1. 系統架構總覽（High-level Architecture）
```
        [Mobile App]
             |
          [API Gateway]
             |
    +---------------------+
    |      QR Service     |  <— Main logic: validate, record, respond
    +---------------------+
         |         |
     [Cache]   [Message Queue]
         |         |
     [Database]  [Log Service, Notification Service]
```
## ⚙️ 2. QR Code 設計與驗證邏輯（核心）
QR Code 格式設計建議（JWT 或 UUID 為 key）：
QR Code 含以下資訊（可 base64 or JWT 編碼）：

uuid: 唯一 ID（避免猜測）

exp: 到期時間戳（如 5 分鐘有效）

event_id: 所屬活動編號

nonce: 隨機數防重放

📌 一次性機制建議：

- 將 uuid 寫入 Redis，TTL(Time To Live) 設為 5 分鐘。

- 使用 Redis 的 SETNX（或 Lua script 保證原子性）來防止重複打卡。

SETNX 是 Redis 的一個指令，意思是：

「SET if Not eXists」→ 只有 key 不存在時，才寫入 value。

🌰 範例：
```
SETNX qr:uuid:abc123 "scanned"
```
如果 qr:uuid:abc123 不存在，就寫入 "scanned"，並回傳 1

如果已經存在，就什麼都不做，回傳 0

這就非常適合：

一次性 QR Code 驗證

Email 驗證碼限制重發

防止重複下單 / 重複打卡   

## 🚀 3. 高併發與低延遲策略（Scaling & Performance）
API Gateway + CDN：前端靜態資源或過期提示可做 edge caching

Rate Limiting / Bot Detection：在 gateway 層進行防濫用設計

Redis + Bloom Filter：在 Redis 加速查詢是否掃描過或過期（減少 DB hit）

* Bloom Filter 是一種 空間效率極高的機率型資料結構，可用來判斷某個元素「是否可能存在」或「一定不存在」。
  - ✨ 特性：可以快速檢查「有沒有看過這個 QR code / user / IP」
  - 記憶體佔用小，可承受數百萬條記錄
  - ✅ 無誤判「不存在」，但 ❌ 可能誤判「存在」
  - 無法刪除元素（但有變種支援）

Message Queue (Kafka/PubSub)：將打卡紀錄 async 寫入 DB，避免同步瓶頸

## 🧮 4. 儲存設計（資料模型與快取）
Redis：
qr:uuid:{uuid} = {user_id, event_id, timestamp}（TTL: 5 mins）

SETNX + Lua 防止 race condition

RDB：
scan_records:

```
(id, user_id, event_id, scan_time, device_info, location)
```
可依 event_id 建 partition 或 index 提升查詢效能

Optional - DynamoDB:
若需要跨地域快速驗證，可使用 DynamoDB + TTL 來替代 Redis（有內建 expiry）

## 🔐 5. 安全性考量
JWT 簽名驗證：防止 tampering
- 他可以解碼 payload 然後改成新的 uuid，但沒有正確的 secret key 就無法重算簽名，因此後端驗證會失敗 → 保護了資料完整性與來源可信度。

IP/device fingerprint + Location check：避免 QR 遭截圖濫用

HTTPS only：避免中間人攻擊

QR code TTL 設短（如 2-5 分鐘）：縮小攻擊窗口

## 🧩 6. Observability & Deployment
Tracing：整合 OpenTelemetry（前端掃描 → API → QR service → Redis/DB）

Logging：記錄每次打卡是否成功、錯誤原因

Prometheus + Grafana：觀察打卡量、錯誤率、處理延遲

Deployment：

Cloud: AWS/GCP ECS + ALB

Auto Scaling：根據 CPU/RPS 彈性調整 QR Service

CI/CD：GitHub Actions + Canary Release

## ⛑ 7. 錯誤處理與 UX 考量
Redis timeout → fallback 提示 “請稍後再試”

錯誤分類處理：

已掃描過：HTTP 409 + message

過期：HTTP 410

格式錯誤 / JWT 驗證失敗：HTTP 400

系統錯誤：HTTP 500

## 測試
- 壓測工具：使用 wrk 或 locust 模擬萬次請求
- 故意中斷過程，觀察 key 是否殘留
- 模擬多使用者撞 UUID，看是否造成 double reward（race condition）
- 
## Question
### 🔐 Q1. 如何在高併發情境下實作 rate limiting？（講 Redis 方案）
✅ 回答範例：
在高併發環境下，我會使用 Redis + atomic operations 實作 rate limiting。這樣既能保證效能，又能防止 race condition。

👣 核心邏輯如下：
go
```
// 用戶 ID/IP 做 key，例如：
key := fmt.Sprintf("rate_limit:%s", userIP)
```
使用 INCR 增加 counter

若是第一次，搭配 EXPIRE 設定時間窗

若 counter 超過閾值，就限流（如 429）

⏱ Lua Script 確保原子性：
```
local current = redis.call("INCR", KEYS[1])
if tonumber(current) == 1 then
  redis.call("EXPIRE", KEYS[1], ARGV[1]) -- 設定 TTL
end
return current
```
這確保了「加 1」和「設 TTL」是原子的，避免 race condition。

🧠 優點：
Redis 是 in-memory，處理速度極快

使用 TTL，自動過期釋放記憶體

高併發下不需頻繁打 DB

📦 延伸：
可以針對 IP、user_id、endpoint 不同層級設不同限制

針對重要服務加權處理（如登入 vs 批量查詢）

### 🤖 Q2. Bot 檢測錯判怎麼處理？會不會誤傷真實用戶？
✅ 回答範例：
是的，這確實是一個必須處理的風險。我會透過「多階層的檢測機制」+「用戶回饋入口」來平衡這個問題。

🧠 我的作法包括：
風險分數制度（Risk Score）
每個請求計算一個 bot 分數（例如：

快速點擊 + UA 可疑 + IP 黑名單 → 得分高

分數超過 80 才 trigger 限制動作（如 CAPTCHA）

非同步封鎖與事後還原
某些封鎖是暫時性的，並允許用戶申請解鎖（例如簡訊驗證、客服復原）

灰名單 vs 黑名單
灰名單用戶會觸發更嚴格限制（如加入滑動驗證或 email 確認）
黑名單才會直接拒絕

✅ 使用者誤傷案例處理：
若判定誤傷，我們會透過驗證碼或補充資訊還原 access

同時記錄這類誤判做調整，持續微調風控模型閾值

### 🙌 Q3. 你如何平衡使用者體驗與安全性？用戶不想一直輸入驗證碼喔？
✅ 回答範例：
這是一個很常見的實務挑戰。為了在體驗與安全之間取得平衡，我會採取「風險感知式安全機制」：

🧠 策略如下：
動態驗證等級（Adaptive Verification）

低風險：不驗證（如登入地點熟悉、UA 正常）

中風險：滑動驗證（無感）

高風險：SMS OTP / Email 驗證

記憶信任設備 / Token

第一次驗證後，發放短期 token 或標記「可信設備」

這樣下次就不需再驗證（除非登入環境變動）

Invisible CAPTCHA (v3)

使用 Google reCAPTCHA v3 評分系統

只有真的太可疑才要求點選交通號誌 😆

行為分析

使用者點擊與滑動習慣可訓練模型區分 bot / 真人

🎯 實務上，我們會進行 A/B Test 來衡量不同驗證策略對轉換率與濫用率的影響，逐步找到最適解。

- OpenTelemetry 或 DataDog 觀察 bot 攻擊流量分佈，讓面試官覺得你有可觀察性概念
- honeypot field（假表單欄位，真用戶不會填，bot 會）是另一種低干擾策略


