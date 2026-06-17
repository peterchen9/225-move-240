# 2026-06-17 .240 Docker port 連線事故紀錄

## 摘要

2026-06-17 檢查 `192.168.16.240` 上三個從 `.225` 搬遷過來的系統時，發現從使用者電腦可以 ping 到 `.240`，SSH `22` 也可連線，但三個 Docker published ports 連不上：

- 場地系統：`http://192.168.16.240:9180/`
- 通訊錄：`http://192.168.16.240:8001/`
- 活石條碼：`http://192.168.16.240:8002/check_in/qr_code/`

最後確認三個系統本身都正常，主要問題在 `.240` 主機的 Docker/iptables 轉送與回程放行規則。

## 影響範圍

受影響的是從 `172.20.0.0/16` 用戶端網段連到 `.240` Docker published ports 的流量。

受影響服務：

- `venue_system-web-1`，外部 port `9180`，容器內 port `80`
- `address_book-web-1`，外部 port `8001`，容器內 port `8000`
- `living_stone_barcode-web-1`，外部 port `8002`，容器內 port `8000`

未受影響或已確認正常：

- `.240` 主機 SSH `22`
- `.240` 主機本機 curl 到 `127.0.0.1:9180`
- `.240` 主機本機 curl 到 `127.0.0.1:8001`
- `.240` 主機本機 curl 到 `127.0.0.1:8002/check_in/qr_code/`

## 現象

從使用者電腦測試：

- `Test-NetConnection 192.168.16.240 -Port 22` 成功
- `curl http://192.168.16.240:9180/` timeout
- `curl http://192.168.16.240:8001/` timeout
- `curl http://192.168.16.240:8002/check_in/qr_code/` timeout

從 `.240` 主機本機測試：

- `http://127.0.0.1:9180/` 回 `302 Found`
- `http://127.0.0.1:8001/` 回 `302 Found`
- `http://127.0.0.1:8002/check_in/qr_code/` 回 `200 OK`

## 根因

`.240` 主機的 Docker/iptables 規則有 Docker bridge isolation/drop 規則，導致外部用戶端連進 Docker published port 後，容器回應封包無法正常回到用戶端。

實際封包觀察：

1. 使用者電腦 `172.20.24.148` 對 `192.168.16.240:9180` 送出 SYN。
2. `.240` 收到 SYN，並 DNAT 到容器 `192.168.80.4:80`。
3. 容器回 SYN-ACK。
4. 回程封包沒有正常離開 `.240` 回到使用者電腦。

因此問題不是網站程式、資料庫或帳密，而是 `.240` 主機層的 Docker/iptables 轉送規則。

## 為何原本可用，後來突然不可用

推定原因是 `.240` 上 Docker 或主機網路規則曾被重建，例如：

- Docker service 重啟
- 主機重開機
- Docker Compose 服務重建
- Docker 更新
- 新增或重建其他 Docker network，導致 iptables/nft 規則順序改變

`.240` 有多張網卡與多個 Docker bridge network，Docker 重新產生 isolation/drop 規則後，published port 的回程轉送比較容易被影響。

## 修正內容

先備份當時防火牆規則：

- `/home/peterchen/iptables_before_docker_port_fix_20260617-085136.txt`

接著在 `.240` 補上 `DOCKER-USER` 最小放行規則，允許 `172.20.0.0/16` 進出三個系統的容器 IP/port。

修正方向包含：

- 用戶端網段到容器的 forward 規則
- 容器回用戶端網段的 return path 規則

## 防止重開後失效

新增 systemd oneshot 服務，讓 Docker 啟動後自動套用這些放行規則。

檔案：

- `/usr/local/sbin/nghcc-docker-port-access.sh`
- `/etc/systemd/system/nghcc-docker-port-access.service`

服務狀態：

- `systemctl is-enabled nghcc-docker-port-access.service`：`enabled`
- `systemctl status nghcc-docker-port-access.service`：`active (exited)`，表示 oneshot 已成功執行

腳本會等待 Docker `DOCKER-USER` chain 和三個容器 IP 出現，再補上規則。這樣容器重建後即使 IP 改變，也會依照容器名稱重新取得新 IP。

## 修正後驗證

從使用者電腦測試：

- `http://192.168.16.240:9180/` 回 `302 Found`
- `http://192.168.16.240:8001/` 回 `302 Found`
- `http://192.168.16.240:8002/check_in/qr_code/` 回 `200 OK`

## 未來若再發生，優先檢查

1. 確認服務是否仍啟用：

   ```bash
   systemctl is-enabled nghcc-docker-port-access.service
   systemctl status --no-pager nghcc-docker-port-access.service
   ```

2. 重新套用規則：

   ```bash
   sudo /usr/local/sbin/nghcc-docker-port-access.sh
   ```

3. 檢查 Docker port 是否仍存在：

   ```bash
   sudo ss -ltnp | egrep ':9180|:8001|:8002'
   ```

4. 檢查 `DOCKER-USER` 規則是否有命中計數：

   ```bash
   sudo iptables -nvL DOCKER-USER
   ```

5. 如果仍 timeout，用 tcpdump 確認封包卡點：

   ```bash
   sudo tcpdump -nn -i any 'host 172.20.24.148 and (tcp port 9180 or tcp port 8001 or tcp port 8002)'
   ```

## 後續風險

未來仍可能再發生的情況：

- 三個容器名稱被改掉
- 用戶端網段不再是 `172.20.0.0/16`
- Docker/iptables/nftables 行為因系統升級而改變
- 有人手動清掉 iptables 規則
- 有人停用或刪除 `nghcc-docker-port-access.service`

目前已加上自動套用機制，一般 Docker 重啟或主機重開後，應可自動恢復。

## 維護原則

依使用者要求，未來場地系統修改以 `.240:9180` 為主，不修改 `.225` 原系統，除非另有明確指示。
