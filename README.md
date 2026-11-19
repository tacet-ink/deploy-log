# tacet.ink 公開部署紀錄（deploy-log）

這個 repo 是 [tacet.ink](https://tacet.ink) 的公開部署帳本，以只增不改（append-only）的慣例維護。每一次正式部署，部署之前先在這裡留下一筆紀錄：當次出貨的版次戳、commit，以及整個 dist 樹每個檔案的路徑與 SHA-256。任何人都可以存檔、監看，並拿線上檔案逐一核對。

English summary: see [English](#english) below.

## 版面

- `deploy-log.jsonl`：根目錄的帳本，一行一件（JSON Lines），`seq` 由 1 起逐行加 1。
- `manifests/<YYYYMMDDTHHMMSSZ>-<版次戳>.json`：每筆部署（或基線）紀錄對應一個 manifest 檔，列出當次 dist 樹的全部檔案。
- `README.md`、`LICENSE.md`：本說明與授權，隨第一筆紀錄一起建立。

## 先記後上

部署流程（tacet 源碼庫的 `deploy.sh`）在建置與出貨面檢查通過之後、Worker 部署之前，先把紀錄推上這個 repo，再以 `git ls-remote` 確認遠端 `main` 逐位等於剛推上的 commit。推不上去，部署就中止。

這個順序有一個需要誠實說明的結果：紀錄寫入之後，部署仍可能在後面的步驟中止（例如 D1 migration 帳本檢查未過，或 Worker 部署步驟失敗）。這時該筆紀錄描述的檔案不會上線，線上與該筆紀錄對不上；修正後的下一次部署會另有一筆新紀錄。所以「有一筆紀錄」的意思是「營運者在那個時刻準備出貨這些檔案」；線上實際提供的是哪一份，以下面的自我驗證為準。

## 帳本行欄位（`deploy-log.jsonl`，schema `tacet-deploy-log/v1`）

| 欄位 | 說明 |
|---|---|
| `seq` | 序號，由 1 起逐行加 1 |
| `schema` | `tacet-deploy-log/v1` |
| `type` | `deploy`（正式部署）、`baseline`（帳本建立時登錄當下的線上版本）、`correction`（更正紀錄，見下） |
| `deployed_at` | 紀錄寫入時刻（UTC，`YYYY-MM-DDTHH:MM:SSZ`）；部署在此之後開始 |
| `version` | 版次戳 `v<套件版本>+<commit 前 7 碼>` |
| `commit` | 建置來源 commit（40 碼） |
| `rb_bundle` | 主程式檔名（`index-*.js`） |
| `rb_hash` | 主程式的 SHA-256（與 tacet.ink/verify 頁「線上現行版次」是同一個值） |
| `index_html` | 根 `index.html` 的 SHA-256 |
| `csp_header` | CSP 標頭值的 SHA-256；`_headers` 沒有 CSP 時為 `null` |
| `manifest` | 對應 manifest 檔在本 repo 內的路徑 |
| `manifest_sha256` | 該 manifest 檔位元組的 SHA-256 |

`baseline` 行的 `deployed_at` 是登錄時刻；被登錄的線上版本，實際部署時刻早於此。

## manifest 欄位（`manifests/*.json`）

| 欄位 | 說明 |
|---|---|
| `schema`、`type`、`deployed_at`、`version`、`commit` | 與帳本行同義 |
| `source_repo` | 建置來源 repo 的 https 網址；無法以公開網址表示時為 `null`。應用層源碼尚未公開，這個連結目前可能需要存取權限 |
| `index_html` | `{path, sha256}` |
| `rb_bundle` | `{file, path, sha256}` |
| `csp_header` | `{source: "_headers", value_sha256}`，或 `null` |
| `csp_header_absent` | `csp_header` 為 `null` 時為 `true` |
| `files` | `[{path, sha256, bytes}]`：dist 樹的全部檔案（含 `_headers`、`_redirects` 等平台設定檔），依路徑排序 |
| `file_count` | `files` 的筆數 |

CSP 雜湊的算法：取 `_headers` 裡 `Content-Security-Policy:` 那一行冒號之後的值，去掉前後空白，以 UTF-8 位元組（不含行尾）計算 SHA-256。

## 為什麼 `manifest_sha256` 記在帳本行上

一個檔案沒辦法在自己的內容裡寫下自己的雜湊：寫進去，雜湊就變了。所以 manifest 檔本身的 SHA-256 記在帳本的那一行，形同一行型的雜湊鏈：帳本行指向 manifest 檔並鎖住它的位元組，manifest 檔再鎖住 dist 樹的每個檔案。`index_html`、`rb_bundle`、`csp_header` 在 manifest 裡有完整的值，帳本行另帶一份，方便快速核對。

## 自我驗證

```sh
# 1. 最新一筆紀錄
curl -s https://raw.githubusercontent.com/tacet-ink/deploy-log/main/deploy-log.jsonl | tail -n 1
# 2. 該行 manifest 欄位所指的檔案；結果應等於該行的 manifest_sha256
curl -s https://raw.githubusercontent.com/tacet-ink/deploy-log/main/manifests/<manifest 檔> | sha256sum
# 3. 線上任一檔案（例：主程式）；結果應等於 manifest 裡同名條目的 sha256
curl -s https://tacet.ink/assets/<bundle 名> | sha256sum
```

線上網址的形狀：`*.html` 以去掉副檔名的網址提供（`privacy.html` 對應 `/privacy`），`<目錄>/index.html` 以 `/<目錄>/` 提供。`_headers`、`_redirects` 是平台設定檔，不會當成網頁提供；`_redirects` 帶 301／308 轉址的來源路徑，線上會轉址而不是回傳檔案本身。

## 更正紀錄

部署完成後，部署流程逐檔比對線上檔案與 manifest（不符時等候數秒重試一次）。重試後仍不符，就追加一筆 `type: "correction"` 的帳本行並告警：

| 欄位 | 說明 |
|---|---|
| `seq`、`schema`、`version`、`commit` | 同上 |
| `type` | `correction` |
| `observed_at` | 觀測到不符的時刻（UTC） |
| `references_seq` | 被更正那筆紀錄的 `seq`（查無時為 `null`） |
| `references_manifest_sha256` | 被更正那筆紀錄的 `manifest_sha256` |
| `base_url` | 比對時的網站根網址 |
| `mismatches` | `[{path, url, expected_sha256, live_sha256 或 http_status}]` |

更正紀錄只追加：既有的行不改寫、不刪除，也不新增 manifest 檔。

## 誠實界限

這些紀錄由營運者寫入。它們的價值是：一段公開、可被第三方存檔與監看的部署歷史。第三方可以核對任何一次部署的檔案與雜湊，也能看出紀錄的中斷或不符。它不等於證明營運者誠實；一個惡意營運者可以停止寫紀錄、寫入假紀錄，也可以改寫這個 repo 的歷史（分支保護能提高門檻，但設定本身也由營運者掌握）。第三方自行存檔或鏡像這個 repo，是讓改寫被看見的方法。

## 授權

本 README 與帳本資料（`deploy-log.jsonl`、`manifests/`）以 CC0 1.0 Universal 釋出至公有領域，見 [LICENSE.md](LICENSE.md)。

## 維運清單

- branch protection：`main` 禁 force-push、禁刪除分支。
- 寫入者是部署流程；人工修正以追加更正紀錄的方式進行。

---

## English

`tacet-ink/deploy-log` is the public, append-only deployment ledger of tacet.ink.

- **Layout**: `deploy-log.jsonl` at the root (JSON Lines, one record per line; `seq` starts at 1 and increases by 1 per line), plus one manifest per deployment under `manifests/`.
- **Record first, deploy second (先記後上)**: before the Worker deployment, the deploy script pushes the record here and checks with `git ls-remote` that the remote `main` matches the pushed commit bit for bit. If the push fails, the deployment aborts. A record can still be followed by an aborted deployment (for example a failed D1 migration check, or a failed Worker deploy step). In that case those files do not go live, the live site does not match that record, and the next deployment adds a new record.
- **Fields**: see the tables above (`seq`, `schema`, `type`, `deployed_at`, `version`, `commit`, `rb_bundle`, `rb_hash`, `index_html`, `csp_header`, `manifest`, `manifest_sha256`). `deployed_at` is the UTC time the record was written; the deployment starts after it.
- **Why `manifest_sha256` lives on the ledger line**: a file cannot contain its own hash, so the manifest's SHA-256 is carried by its ledger line (a one-line hash chain). The manifest pins every file in the shipped dist tree.
- **Self-verify**: run the three `curl … | sha256sum` commands above and compare.
- **Corrections**: if a live file differs from its manifest after deployment (after one retry), a `type: "correction"` line is appended and an alert is raised. Existing lines are not rewritten.
- **Honest limits**: these records are written by the operator. Their value is a public deployment history that third parties can archive and watch. They do not prove that the operator is honest: a malicious operator can stop writing records, write false ones, or rewrite this repo's history. Independent archives and mirrors are how such rewrites become visible.
- **License**: this README and the ledger data are released under CC0 1.0 Universal (public domain dedication); see LICENSE.md.
- **Maintenance**: branch protection on `main`: no force-push, no branch deletion.
