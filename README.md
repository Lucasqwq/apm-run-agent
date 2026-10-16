# APM RUM Agent 公網接入 SOP（ALB + ACM，ap-east-1）

> **公開版**：主機名、IP、instance ID、帳號名已改成占位符。內部重建請用內部 SOP 的實際值。
> **狀態**：2026-10-15 已拆除（ALB / Target Group / ACM / DNS / 測試頁均已移除）。本文為**留存**，日後要重啟前端 RUM 時照此重建。  
> **驗證日期**：2026-10-14，Kibana 可見服務 `prod-apple-channel`、事務類型 `page-load`。  
> **前置**：ELK 叢集與 APM Server 已依內部建置手冊建好（APM Server 9.4.3，lka-01 / lka-02 的 `:8200`）。這份公開文件不含那份手冊。

---

## 目錄

- [0. 這份在做什麼](#0-這份在做什麼)
- [1. 架構](#1-架構)
- [2. APM Server 啟用 RUM（lka-01 / lka-02）](#2-apm-server-啟用-rumlka-01--lka-02)
- [3. AWS：ACM 憑證 + DNS](#3-awsacm-憑證--dns)
- [4. AWS：Target Group + ALB + 安全群組](#4-awstarget-group--alb--安全群組)
- [5. 前端嵌入 RUM Agent](#5-前端嵌入-rum-agent)
- [6. 驗證](#6-驗證)
- [7. 防護（公網入口）](#7-防護公網入口)
- [8. 費用](#8-費用)
- [9. 拆除清單](#9-拆除清單)
- [10. 踩雷筆記](#10-踩雷筆記)
- [11. 觀念補充](#11-觀念補充)

---

## 0. 這份在做什麼

後端 Python APM Agent 只能看到請求**進到 Python 之後**的事務（例如 `apple-app` 的 `GET …YunxinGroupListView` 39 ms）。使用者端的 **DNS / TCP / TLS / 整頁載入**發生在那之前，後端看不到。

Elastic **RUM Agent**（`@elastic/apm-rum`）跑在瀏覽器裡，讀 Navigation Timing 另開一筆 `page-load` 事務，把 **Domain lookup、TCP、Request、Response、Processing** 拆開顯示。資料一樣送進現有 APM Server → Elasticsearch → Kibana。

瀏覽器連不到內網的 APM NLB，而且頁面是 `https://`，所以需要一個**對外 HTTPS 入口**（ALB + ACM），再轉到 lka 的 HTTP `8200`。

---

## 1. 架構

```text
使用者瀏覽器（https://<test-page-domain> 頁面內的 RUM script）
    │  POST https://<apm-domain>/intake/v2/rum/events
    ▼
<apm-domain>  ── CNAME ──▶  <alb-dns-name>
    │  internet-facing ALB · HTTPS:443 · ACM(<apm-domain>)
    ▼
Target Group lka-rum-agent（HTTP 8200，health check "/"）
    ├── lka-01  <lka-01-private-ip>:8200
    └── lka-02  <lka-02-private-ip>:8200
    │
    ▼
APM Server → Elasticsearch :9200（<es-writer-user>）→ Kibana Observability → Applications
```

業務機（Python / Node agent）**不變**，繼續走內網 `prod-lka-apm-nlb-…:8200`。這台 ALB 只給瀏覽器。

| 元件 | 值（2026-10-14 實際） |
|---|---|
| ALB 名稱 | `lka-rum-agent`（internet-facing，3 個 AZ） |
| ALB DNS | `<alb-dns-name>` |
| 對外網域 | `<apm-domain>`（CNAME → ALB DNS） |
| ACM 憑證 | `<apm-domain>`，ap-east-1，DNS 驗證 |
| Target Group | `lka-rum-agent`，Instance，HTTP 8200，health `/` |
| Targets | `<lka-01-instance-id>`、`<lka-02-instance-id>`（lka-01 / lka-02） |
| RUM serviceName | `prod-apple-channel` |
| 測試頁 | `https://<test-page-domain>`（nginx `download_apple-test` / `/data/www/downloadpages-test`） |

---

## 2. APM Server 啟用 RUM（lka-01 / lka-02）

**兩台都要改**，NLB / ALB 會輪流打到。

`/etc/apm-server/apm-server.yml`，在既有 `apm-server:` 區塊補上 `rum:`：

```yaml
apm-server:
  host: "0.0.0.0:8200"
  max_event_size: 512000
  worker: 16
  bulk_max_size: 8192
  read_timeout: 30s
  write_timeout: 30s
  rum:
    enabled: true
    allow_origins: ['*']
```

- `allow_origins: ['*']` 放行所有網頁來源（也是預設值）。測試站、正式站都不用逐條加。只影響瀏覽器 CORS；`curl` 直接 POST 不看這個。
- 公開入口用 `*` 的前提是 ALB 已經只轉發 RUM 路徑，而且 APM Server 設了 `allow_service`（見 §7）。否則任何網站都能把事件寫進 Elasticsearch。
- 若之後 agent 開 `sendCredentials`，`*` 會失效，必須改成明確 `https://網域`（不含 path；443 不用寫 port）。
- 要收緊時可改成：`allow_origins: ['https://<test-page-domain>', 'https://*.example.com']`。

```bash
sudo chown root:apm-server /etc/apm-server/apm-server.yml
sudo chmod 640 /etc/apm-server/apm-server.yml
sudo systemctl restart apm-server      # 注意拼字，不是 apm-serveer
sudo systemctl status apm-server
curl -s http://127.0.0.1:8200/          # 期望回 version JSON
```

`apm-server` 是 deb 套件自帶的 systemd 服務（`/usr/lib/systemd/system/apm-server.service`），不用另外寫 unit。

---

## 3. AWS：ACM 憑證 + DNS

HTTPS listener **一定要**憑證。ALB 預設的 `*.elb.amazonaws.com` 不能申請憑證，也不能直接當 `serverUrl`（頁面是 https，瀏覽器會判憑證不符）。

1. **ACM（ap-east-1）** → Request public certificate → 網域填 `apm.<你的網域>`（例：`<apm-domain>`）→ DNS 驗證。
2. 到 DNS 加 ACM 給的驗證 CNAME，等狀態變 **Issued**。
3. ALB 建好後，再加一筆 CNAME：`apm.<你的網域>` → `<ALB DNS>`。

憑證網域**不必**跟網頁網域相同；只要跟 RUM `serverUrl` 的主機名一致即可。

---

## 4. AWS：Target Group + ALB + 安全群組

### 4.1 Target Group

| 欄位 | 值 |
|---|---|
| Name | `lka-rum-agent` |
| Target type | Instance |
| Protocol / Port | **HTTP / 8200**（不是 HTTPS 443） |
| Protocol version | HTTP1 |
| VPC | lka 所在 VPC（`<lka-vpc-cidr>`） |
| Health check | HTTP，path `/`，port = traffic port，成功碼 200 |
| Targets | lka-01、lka-02 |

### 4.2 ALB

| 欄位 | 值 |
|---|---|
| Type | **Application Load Balancer**（不是 NLB；要卸 TLS、要按路徑過濾） |
| Scheme | internet-facing |
| AZ | 2～3 個 |
| Listener | **HTTPS 443**，Default SSL cert 選 ACM `apm.<網域>` |
| Default rule | Forward → `lka-rum-agent`（正式建議改成只放行 RUM 路徑，見 §7） |

### 4.3 安全群組（最常卡的地方）

| 掛在哪 | 入站 | 來源 |
|---|---|---|
| **ALB 的 SG** | TCP **443** | `0.0.0.0/0`（或 CDN 回源段） |
| **lka 的 SG**（instance 原本那張） | TCP **8200** | **ALB 的 SG ID** |

- 8200 不要加在 ALB 的 SG 上；443 不要加在 lka 上。
- 是在 lka 既有 SG 上**加一條入站**，不是把 ALB 的 SG 掛到 instance。
- lka 原本的 SG 保留（業務機走 internal NLB 的 8200 靠它）。
- 改完等 1～2 分鐘，Target health 由 `Unhealthy: Request timed out` 轉 `healthy`。

### 4.4 為什麼不是 NLB

| | NLB | ALB |
|---|---|---|
| 看得懂 HTTP 路徑 | 否 | 是（可只放行 `/intake/v2/rum/*`） |
| 卸 HTTPS → 轉 HTTP 8200 | 需另建 TLS listener | 原生 |
| 適用 | Filebeat 5044、內網 agent 8200 這種純 TCP 轉發 | 瀏覽器 HTTPS 入口 |

現有 internal NLB 繼續服務業務機，不動。

---

## 5. 前端嵌入 RUM Agent

本節的目錄與 nginx 片段名稱（`download_apple`、`$devtools_guard_tag` 等）是這次測試站自己的設定。要看的是做法：頁面 `<head>` 注入 RUM script，`serverUrl` 指到對外的 HTTPS 網域。

### 5.1 靜態下載頁（無前端工程師也能做）

下載頁是打包好的 Vue 靜態檔，在 `prod-web-cdn` 的 `/data/www/downloadpages/index.html`（多個渠道域名**共用**此目錄）。測試時另複製一份：

```bash
# prod-web-cdn
cp -a /data/www/downloadpages /data/www/downloadpages-test
cp /etc/nginx/tmp/download_pages/download_apple /etc/nginx/tmp/download_pages/download_apple-test
# download_apple-test 內 root 改為 /data/www/downloadpages-test
# conf.d/channel/apple/<test-page-domain>_261013.conf 改 include tmp/download_pages/download_apple-test
nginx -t && nginx -s reload
```

`index.html` 的 `<head>` 第一個 script 放：

```html
<head><script src="https://unpkg.com/@elastic/apm-rum/dist/bundles/elastic-apm-rum.umd.min.js"></script><script>
elasticApm.init({
  serviceName: 'prod-apple-channel',
  serverUrl: 'https://<apm-domain>',
  environment: 'production',
  breakdownMetrics: true
})</script><meta charset="utf-8">…
```

- `serviceName` 用 `prod-{業務}-{服務名}`，**不要**和後端 `apple-app` 同名。
- `serverUrl` 填 ACM 憑證那個網域，**不帶 port**、不要填 `elb.amazonaws.com`、不要填內網 NLB、不要填 `http://`。
- nginx 的 `sub_filter '<head>' '<head>$devtools_guard_tag'` 仍會在最前面插 F12 腳本，順序為 guard → RUM，可以。
- 正式前端若有打包流程，改在入口第一行 `import { init } from '@elastic/apm-rum'`，並加 `distributedTracingOrigins: ['https://<API 網域>']`，瀏覽器打 API 時會帶 `traceparent`，Kibana 能把 page-load 和後端 `apple-app` 事務串成同一條 trace。

### 5.2 H5（apple-nodenuxt 反代）

H5 的 `location @spa` 已有 `proxy_set_header Accept-Encoding "";` + `sub_filter '<head>' '<head>$devtools_guard_tag';`，直接把 RUM `<script>` 串進 `sub_filter` 的替換內容即可，不需前端重新打包。

---

## 6. 驗證

1. 瀏覽器開 `https://<test-page-domain>`（**用網域**，用 IP 不會查 DNS）。
2. DevTools → Network：`OPTIONS https://<apm-domain>/intake/v2/rum/events` → **200**；`POST` 同路徑 → **202 Accepted**。
3. 等 1～2 分鐘 → Kibana（`<kibana-url>`）→ Observability → Applications → `prod-apple-channel`。
4. 事務類型選 **page-load**，點一筆樣本：時間線依序為 **Domain lookup → TCP → Requesting and receiving the document → Parsing → 資源**。

- 事務名會是 `/?{query}`：網址帶 `?auth_token=…`，agent 把查詢字串收斂成 `{query}`。
- **Domain lookup 為 0 / 不顯示**：瀏覽器已快取此網域 DNS。`chrome://net-internals/#dns` 清 host cache 再強制重整。只有整頁首次載入會算 DNS；SPA 換頁與 XHR 不會。
- 本機探測：

```bash
curl -sv --max-time 12 -o /dev/null \
  -w "dns:%{time_namelookup} connect:%{time_connect} tls:%{time_appconnect} ttfb:%{time_starttransfer} code:%{http_code}\n" \
  https://<apm-domain>/
```

---

## 7. 防護（公網入口）

RUM 入口設計上**不驗 token**（瀏覽器藏不住密鑰），`allow_origins` 只是 CORS，`curl` 帶假 `Origin` 一樣能 POST。防線放在「縮小入口 + 限制服務名 + 單 IP 限流」：

### 7.1 ALB 只放行 RUM 路徑

Listener rules：
- 新規則：Path `/intake/v2/rum/*`、`/config/v1/rum/*` → Forward `lka-rum-agent`
- Default rule：Fixed response **403**

這樣 `/`（版本 JSON）與後端專用 `/intake/v2/events` 都不會露到外網。

### 7.2 APM Server 匿名限制

```yaml
apm-server:
  auth:
    anonymous:
      enabled: true
      allow_agent: [rum-js, js-base]
      allow_service: ["prod-apple-channel", "prod-apple-h5"]
      rate_limit:
        ip_limit: 1000
        event_limit: 30
  rum:
    enabled: true
    allow_origins: ['*']
```

> 若同時啟用 `auth.secret_token`，後端 agent 必須帶 token，RUM 仍可匿名。目前後端 agent 未帶 token，啟用前要先全部補上。

### 7.3 CDN（廠商）

- `apm.<網域>` CNAME 改指 CDN，CDN 回源到 ALB DNS。
- CDN 上對 `POST /intake/v2/rum/events` 設 **CC / 單 IP 頻率限制**，超過直接丟或回 429，不回源。
- **ALB 的 SG 入站 443 改成只放行 CDN 回源 IP 段**，否則別人直連 ALB DNS / IP 仍可繞過。
- CDN 的用途：擋量（也省 ALB LCU 費）、隱藏 ALB 真實位址、改善跨區握手。單純為了不讓 CNAME 顯示負載均衡器名稱不值得加。

---

## 8. 費用（ap-east-1，2026-09/10 價目表）

重建前請重查當時的 AWS 價目表。下面是 2026-09/10、ap-east-1 的紀錄，單價會變。

| 項目 | 單價 | 月（730h） |
|---|---|---|
| ALB | $0.0277 / hr | ≈ $20.2 |
| 公開 IPv4 | $0.005 / 個 / hr | 2 AZ ≈ $7.3；3 AZ ≈ $11.0 |
| LCU | $0.0088 / LCU-hr | RUM 低流量 ≈ $1～6 |
| ACM 公開憑證、安全群組 | $0 | $0 |

**合計約 $28～38 / 月**（3 AZ 約 $33）。對照自建 Sentry 單機（4C16G + 100 GB gp3 ≈ $178；4C32G ≈ $222），每月約省 $145～190。

---

## 9. 拆除清單（2026-10-15 已執行）

重建時反向照做；拆除時依序：

1. 前端：`index.html` 移除 RUM `<script>`；nginx `<test-page-domain>_*.conf` 改回 `download_apple`；刪 `download_apple-test`、`/data/www/downloadpages-test`；`nginx -t && nginx -s reload`。
2. DNS：刪 `<apm-domain>` CNAME 與 ACM 驗證 CNAME。
3. AWS：刪 ALB `lka-rum-agent` → 刪 Target Group `lka-rum-agent` → 刪 ALB 的 SG → 在 lka SG 移除「8200 來源 ALB SG」那條 → 刪 ACM 憑證 `<apm-domain>`。
4. APM Server：`rum.enabled` 可保留（內網 NLB 不對外，無害）；要關就兩台都改回並 `systemctl restart apm-server`。
5. Kibana 內 `prod-apple-channel` 的歷史資料會隨 ILM 到期自然清除。

---

## 10. 踩雷筆記

| 現象 | 原因 / 解法 |
|---|---|
| `Unit apm-serveer.service could not be found` | 拼字；服務名是 `apm-server` |
| RUM console `Failed sending events! AbortError: signal is aborted without reason` | 瀏覽器連 `apm.<網域>:443` 逾時（約 10 s 後 agent 自中止）。查 ALB SG 443 是否放行、DNS 是否指到 ALB |
| Target `Unhealthy: Request timed out`，但 lka 上 `curl 127.0.0.1:8200` 正常 | lka SG 沒放行 **8200 來源 = ALB SG**；或 port 誤填（曾打成 0） |
| Target Group 建成 HTTPS 443 | 要 **HTTP 8200**；443 只在 ALB listener |
| `serverUrl` 填 `https://prod-lka-logstash-nlb…:8200` | 指錯台（那是 Logstash NLB）；且 8200 上沒有 TLS |
| `serverUrl` 填 `http://…` 或 `elb.amazonaws.com` | https 頁面會擋 http；ELB 預設名無憑證 |
| ALB SG 開了 443 + 8200 | 8200 不用在 ALB；ALB 只需 443 |
| `allow_origins` 寫 `https://<test-page-domain>:443` | 443 是預設 port，不能寫；8200 也不是這裡 |
| 本機 `curl` ALB 被 sandbox proxy 403 | 本機代理，與 AWS 無關；換網路或關代理再測 |
| Kibana 搜 `prod-apple-chanel` 找不到 | 少一個 `n`；要跟 `serviceName` 完全一致 |
| 時間線沒有 Domain lookup | DNS 已快取為 0；清 host cache 後強制重整 |
| lka 上 `curl` 自己的 NLB `:8200` 不通 | NLB 不支援 hairpin；在業務機測，lka 上用 `localhost:8200` |

---

## 11. 觀念補充

- **一次 API 呼叫的時間** = DNS → TCP → TLS →（送出）→ 等第一個 byte → 收 body。前三段是連線階段，有連線池 / keep-alive 時常為 0。
- **Metrics vs Tracing**：Grafana 曲線是很多請求的統計；tracing 是**單一次**請求的時間軸（trace 內多個 span），跨服務靠 header 帶 trace id（W3C `traceparent`）。
- **Elastic 名詞**：進到服務那層叫 **transaction**（Kibana 中文「事務」），裡面的 Redis / SQL / HTTP 叫 **span**。
- **APM Server vs Sentry**：都是 tracing 接收端。APM Server 寫 ES、Kibana 看；Sentry 自家儲存，強在錯誤分群 / release / 回放。前端 page-load / DNS 兩邊都能量，只做一邊即可。
- **RUM 能不能取代 Sentry**：前端效能這段可以；錯誤 issue 化、版本歸因、Session Replay 不能。
- **APM Server 與 ES / Kibana 關係**：APM Server `output.elasticsearch` 直寫 ES（`<es-writer-user>`），Kibana 只查詢；`xpack.fleet.packages: apm` 只是裝畫面。
- **Internal LB 連不到**不是因為跨帳號，是沒有網路路徑（無 peering / TGW / VPN）；同帳號跨 VPC 也一樣。
