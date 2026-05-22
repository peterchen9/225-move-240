# 225 搬家到 240 專案紀錄

更新日期：2026-05-22

## 主機資訊

- 舊主機：`192.168.16.225`
- 新主機：`192.168.16.240`
- SSH：`peterchen / nldwehdw`

## 新系統網址

- 場地系統：`http://192.168.16.240:9180/`
- 通訊錄系統：`http://192.168.16.240:8001/contacts/`
- 活石條碼系統：
  - 掃碼報到：`http://192.168.16.240:8002/check_in/qr_code/`
  - 名字或電話報到：`http://192.168.16.240:8002/check_in/phone_num/`
  - 列印條碼：`http://192.168.16.240:8002/check_in/ask_print/`
  - 近三分鐘報到名單：`http://192.168.16.240:8002/check_in/recent/`
  - 報到紀錄後台：`http://192.168.16.240:8002/check_in/admin/check_in/checkinrecord/`

## 共用帳號

三個新系統都已新增：

- `admin / admin`

另外已設定：

- 場地系統：`peterchen / nldwehdw`
- 通訊錄系統：`peterchen / nldwehdw`
- 活石條碼系統：`peterchen / nldwehdw`

## 場地系統處理紀錄

路徑：`/home/peterchen/venue_system`

主要處理：

- 將舊 MRBS 介面與資料搬到 Docker。
- 修正缺少 `config.inc.php` 的錯誤。
- 改用舊系統主畫面版型。
- 修正 PHP 8 相容問題。
- 修正月曆只顯示星期、不顯示日期的問題。
- 將說明頁英文翻譯成中文。
- 新增帳號 `peterchen / nldwehdw`。
- 新增帳號 `admin / admin`。

狀態：

- `http://192.168.16.240:9180/` 可開。
- 場地日曆、登入、說明頁已測過。

## 通訊錄系統處理紀錄

路徑：`/home/peterchen/address_book`

主要處理：

- 將新版面改回舊系統風格。
- 修正 Python 2 語法到 Python 3。
- 修正 Django 3 相容問題。
- 補上 `/contacts/...` 舊網址相容。
- 修正照片不顯示問題。
- 改成讀正確 media/static 目錄。
- 移除 e-mail、電話、簡訊等固定小圖，只保留個人照片與家庭/團體照。
- 修正舊站 `app.nghcc.org.tw` 連結，改成新系統 `/contacts/...`。
- 修正首頁三個入口：
  - 通訊錄匯入系統
  - 新朋友輸入及資料匯出系統
  - 通訊錄搜尋修改系統
- 新增帳號 `peterchen / nldwehdw`。
- 新增帳號 `admin / admin`。

已測：

- `http://192.168.16.240:8001/contacts/`
- `http://192.168.16.240:8001/contacts/add/1/`
- `http://192.168.16.240:8001/contacts/imports/1/`
- `http://192.168.16.240:8001/contacts/modify/1/`
- 搜尋 `0932285036` 可找到 `陳潘傳`。
- 修改頁照片可顯示。

## 活石條碼系統處理紀錄

路徑：`/home/peterchen/living_stone_barcode`

主要處理：

- 確認三個核心畫面：
  - 掃碼報到
  - 名字或電話報到
  - 列印條碼
- 修正 `/static/...` 靜態檔 404，背景圖與 `md5.min.js` 現在可載入。
- 修正通訊錄資料庫路徑，改成讀新通訊錄系統同一份 `ContactsDB`。
- 修正列印流程內舊連結：
  - `127.0.0.1:8000`
  - `192.168.16.225:8008`
- 新增 Django admin 帳號：
  - `peterchen / nldwehdw`
  - `admin / admin`
- 設定報到紀錄後台。
- 修正近三分鐘報到名單：
  - 改用 `timezone.now()`
  - 同一位兄姊只顯示最新一筆

已測：

- `http://192.168.16.240:8002/check_in/qr_code/`
- `http://192.168.16.240:8002/check_in/phone_num/`
- `http://192.168.16.240:8002/check_in/ask_print/`
- `http://192.168.16.240:8002/check_in/recent/`
- `http://192.168.16.240:8002/check_in/admin/check_in/checkinrecord/`
- 電話 `0932285036` 可查到 `陳潘傳`。
- QR ID `NG000015718` 可報到。
- 報到紀錄資料表目前約 `130304` 筆。

注意：

- 實體列印底層仍是舊系統的 Windows `EZio64.dll` + USB 標籤機程式。
- 目前 Docker 是 Linux 容器，不能直接執行 Windows DLL。
- 列印前的查詢與確認頁已可用；真正按下「是，請列印」需要另外處理列印層或改成可在 Linux/Docker 使用的列印方式。

## 仍需留意

- `admin/admin` 是弱密碼，測試完成後建議改成較安全密碼。
- 若通訊錄資料更新，活石條碼目前已掛載同一份 `ContactsDB`，理論上會同步讀到。
- 活石條碼的實體標籤機列印是目前最大剩餘風險。
