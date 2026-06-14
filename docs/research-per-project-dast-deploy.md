# Nghiên cứu: triển khai DAST + Deploy cho từng project tích hợp

> Branch: `research/per-project-dast-deploy`. Mục tiêu: thiết kế cách mỗi repo
> tích hợp `sast-action` bật được **deploy (CD)** và **DAST (ZAP)** theo project,
> và để `chat-system` theo dõi/ingest kết quả per-project. Dựa trên đọc code thật
> ở cả `sast-action` (master) và `chat-system` (`ft/imp-fe`). Ngày: 2026-06-05.

---

## 1. Câu hỏi nghiên cứu
1. Mỗi project tích hợp bật DAST + deploy **bằng cách nào**?
2. Kết quả deploy/DAST chảy về `chat-system` per-project ra sao?
3. Còn thiếu gì để trải nghiệm "drop 1 block là chạy" đúng cho multi-project?

**Kết luận sớm:** phần lớn **đã chạy được**. DAST đi trọn vòng (ZAP → artifact →
ingest thành findings). Deploy chạy phi tập trung (mỗi repo tự deploy qua Render
hook). Việc còn lại chủ yếu là **trám lỗ hổng cấu hình + quan sát**, không phải
viết lại từ đầu.

---

## 2. Hiện trạng (đã kiểm chứng)

### 2.1 Phía `sast-action` — đã có job `cd` + `dast`
Reusable `sast-ci.yml` đã wire sẵn (chỉ chạy khi caller bật input):

| Job | Bật khi | Làm gì | Artifact |
|---|---|---|---|
| `cd` | `deploy:true` & `sast` success & `gate` success/skipped | `build-image` (docker build+push, Trivy image scan) + `deploy-staging` (Render deploy hook) | `trivy-image-scan-<run>` (SARIF) |
| `dast` | `dast:true` & `staging_url!=''` & `cd` success | `sleep 60` (chờ Render roll) → `run-dast` (ZAP baseline/full) → notify `dast_complete` | `dast-reports-<run>` (zap.json/md/html) |

Chi tiết:
- `build-image` (`actions/build-image/action.yml`): tag `<image_repo>:<sha>` + `:latest`, output `image_tag`, login Docker Hub khi `push=true`.
- `deploy-staging` (`actions/deploy-staging/action.yml`): `curl POST <render_deploy_hook>` — **fire-and-forget** (`wait_seconds=0`, không poll trạng thái Render).
- `run-dast` (`actions/run-dast/action.yml`): `zaproxy/action-baseline@v0.13.0` (hoặc `action-full-scan@v0.12.0`), output `reports/zap.json|md|html`, `fail_on_warn=false`, `allow_issue_writing=false`.

**Per-project = đã có sẵn về bản chất**: mỗi repo gọi reusable với `with:` +
`secrets:` riêng → tự deploy + DAST bằng config của mình.

### 2.2 Phía `chat-system` (`ft/imp-fe`) — ingest + monitor

| Thành phần | Trạng thái | Ghi chú |
|---|---|---|
| `Project.staging_url` | ✅ có (V3.7) | URL để monitor ping; "thường trùng staging_url truyền cho DAST" |
| Monitor uptime | ✅ đầy đủ | ping `staging_url` mỗi project, alert `down`/`recovered` (ngưỡng `MONITOR_DOWN_THRESHOLD`) |
| **Ingest DAST** | ✅ **đầy đủ** | `ZapJsonNormalizer` (`normalizer.py`) parse `dast-reports-*.json`, map risk 0–3→info/low/med/high, promote high→critical cho CWE injection/RCE, dedup (rule_id,uri,alert); profile có prefix `dast-reports-` |
| DAST trong stats/AI | ✅ | `stats_service` đếm `dast_total`, `dast_critical_high`; có trong AI analysis/report |
| **Gate theo DAST** | ❌ thiếu | security-gate chỉ đếm tool SAST; finding `owasp-zap` KHÔNG chặn pipeline |
| Trigger deploy từ MCP | ⚠️ stub | `GitHubClient.dispatch_workflow()` có, nhưng chỉ qua MCP resource — **chưa có REST endpoint** |
| Per-project deploy config | ❌ thiếu | `Project` không có field `render_url`/`docker_image`/`deploy_workflow` |
| Alert `deploy_failed` | ⚠️ stub | có trong enum comment, chưa code nào raise |
| Integration snippet | ⚠️ một phần | `/projects/{id}/integration` chỉ sinh block webhook SAST — **không nhắc DAST/deploy/staging_url** |

---

## 3. Quyết định kiến trúc: phi tập trung (khuyến nghị) vs tập trung

| | **A. Phi tập trung — observe** (khuyến nghị) | **B. Tập trung — orchestrate** |
|---|---|---|
| Ai deploy/DAST | Workflow của từng repo (qua reusable) | `chat-system` gọi `workflow_dispatch`/Render API |
| Secrets | Ở mỗi repo (docker_*, render hook) — không rời repo | Tập trung ở MCP DB (phải mã hoá — đã có `FERNET_KEY`) |
| chat-system | Chỉ **ingest + monitor + hiển thị** | Thêm REST trigger + lưu deploy config + theo dõi trạng thái deploy |
| Khớp thiết kế hiện tại | ✅ đúng nguyên trạng | ⚠️ cần build mới nhiều |
| Bảo mật | ✅ tốt (secret phân tán theo repo) | ⚠️ MCP nắm credential deploy của mọi project |
| Hợp đồng tốt nghiệp | Đủ mạnh để demo DevSecOps đa-project | "wow" hơn nhưng rủi ro/khối lượng lớn |

→ **Khuyến nghị A**: giữ deploy/DAST ở mỗi repo (reusable workflow), `chat-system`
đóng vai **observer** (ingest DAST findings, monitor uptime, hiển thị trạng thái).
Đây là mô hình đang chạy; chỉ cần trám lỗ hổng (§5). B chỉ làm khi cần demo
"deploy từ dashboard".

---

## 4. Recipe bật DAST + Deploy cho 1 project (mô hình A)

### 4.1 Trong caller workflow của repo (vd `ci.yml`)
```yaml
jobs:
  security:
    uses: cochecheee/sast-action/.github/workflows/sast-ci.yml@master
    with:
      language: java
      # ---- Deploy (CD) ----
      deploy: true
      image_repo: cochecheee/<repo>          # Docker Hub repo
      dockerfile: Dockerfile
      build_context: .
      # ---- DAST ----
      dast: true
      staging_url: https://<repo>.onrender.com   # URL ZAP quét
      dast_scan_type: baseline                   # baseline | full
    secrets:
      dashboard_url:      ${{ secrets.MCP_GATEWAY_URL }}
      dashboard_token:    ${{ secrets.MCP_WEBHOOK_TOKEN }}
      docker_username:    ${{ secrets.DOCKER_USERNAME }}
      docker_password:    ${{ secrets.DOCKER_PASSWORD }}
      render_deploy_hook: ${{ secrets.RENDER_DEPLOY_HOOK }}
```
Luồng: `sast → gate → cd (build+push+deploy) → dast (ZAP)`. Gate fail → không
deploy/DAST. DAST artifact `dast-reports-<run>` được poller MCP kéo về ingest.

### 4.2 Trong `chat-system` (per-project)
- Tạo Project, set `staging_url` (qua `/projects/{id}/monitor` hoặc UI) = đúng URL
  DAST → monitor uptime tự bật.
- (Tuỳ chọn) set `mcp_project_id` ở caller để bật learning-loop gate cho SAST.

→ Với mô hình A, **không cần thêm secret nào ở chat-system** — deploy chạy bằng
secret của repo.

---

## 5. Khoảng trống & đề xuất (ưu tiên)

### G1 — `staging_url` đang phải khai 2 nơi (⭐ ưu tiên cao, dễ)
Hiện caller khai `staging_url` (cho ZAP) và Project.staging_url (cho monitor) tách
rời → dễ lệch. **Đề xuất**: `notify-dashboard` gửi kèm `staging_url` trong payload
webhook DAST; MCP tự cập nhật `Project.staging_url` nếu trống. Một nguồn sự thật.
- Sửa: `actions/run-dast`/`notify-dashboard` thêm field; MCP webhook handler nhận.

### G2 — Integration snippet thiếu DAST/deploy (⭐ cao, dễ)
`/projects/{id}/integration` chỉ sinh block SAST. **Đề xuất**: thêm phần tuỳ chọn
DAST + deploy (đúng §4.1) để người dùng copy-paste là bật được.
- Sửa: `mcp/src/api/projects.py` (hàm sinh snippet).

### G3 — DAST chưa được gate (trung bình, quyết định chính sách)
Finding `owasp-zap` không chặn pipeline. **Đề xuất** (tuỳ chính sách):
- Thêm input `gate_on_dast: false` (mặc định) ở reusable; khi bật, một job
  `dast-gate` đếm ZAP high/critical từ `dast-reports-` và fail nếu vượt ngưỡng.
- Hoặc phía MCP: mở rộng `/findings/gate-count` nhận `?include_dast=1`.
- Lưu ý: DAST thường chạy **sau** deploy → gate DAST không chặn được deploy của
  chính run đó; phù hợp làm "post-deploy gate" (chặn promote prod ở run sau).

### G4 — Deploy thiếu quan sát: không poll trạng thái Render (trung bình)
`deploy-staging` fire-and-forget → không biết deploy fail. **Đề xuất**:
- `deploy-staging` poll Render API tới khi `live`/`failed` (cần `RENDER_API_KEY`),
  set output `deploy_status`; notify MCP `pipeline-status=deploy_failed` khi lỗi.
- MCP: raise `Alert(kind="deploy_failed")` (enum đã có sẵn, chỉ thiếu code raise).

### G5 — (Tuỳ chọn, mô hình B) trigger deploy từ dashboard (cao, lớn)
Nếu muốn "deploy từ UI": thêm REST `POST /projects/{id}/deploy` gọi
`GitHubClient.dispatch_workflow()` (đã có) hoặc Render API; thêm field deploy
config + mã hoá bằng `FERNET_KEY`. **Chỉ làm nếu thực sự cần** — đi ngược mô hình A.

---

## 6. Kế hoạch triển khai đề xuất (theo phase)

| Phase | Nội dung | Gaps | Quy mô |
|---|---|---|---|
| P1 | Tài liệu + recipe bật DAST/deploy per-project; demo 1 repo (vd ALOUTE) bật `dast:true` | — | nhỏ |
| P2 | Snippet integration sinh kèm DAST/deploy; tự sync `staging_url` qua webhook | G1, G2 | nhỏ |
| P3 | Quan sát deploy: poll Render + alert `deploy_failed` | G4 | vừa |
| P4 | (Tuỳ chọn) DAST gate post-deploy | G3 | vừa |
| P5 | (Tuỳ chọn) deploy từ dashboard (mô hình B) | G5 | lớn |

Làm P1–P2 trước là đủ cho một demo multi-project hoàn chỉnh.

---

## 7. Rủi ro / lưu ý
- **Render free tier**: container sleep → ZAP có thể quét lúc app chưa wake; `sleep 60`
  hiện tại có thể chưa đủ. Cân nhắc health-check chờ `200` trước khi ZAP.
- **DAST tốn thời gian**: `full` scan ~30 phút; mặc định `baseline` (~5 phút) cho CI.
- **Secret sprawl (mô hình A)**: mỗi repo cần docker_* + render hook. Tài liệu hoá rõ.
- **Idempotency ingest**: poller đã dedup theo run_id + artifact; DAST normalizer dedup
  theo (rule_id,uri,alert) → an toàn khi reprocess.
- **Gate vs DAST timing**: DAST sau deploy nên không thể "chặn trước khi deploy" trong
  cùng run — định vị là cổng cho lần promote sau.

---

## 8. Tham chiếu (file:line)
**sast-action**
- `.github/workflows/sast-ci.yml` — job `cd` (~209–248), `dast` (~255–288)
- `actions/build-image/action.yml`, `actions/deploy-staging/action.yml`, `actions/run-dast/action.yml`

**chat-system (`ft/imp-fe`)**
- `mcp/src/models/entities.py` — `Project.staging_url` (~68), Alert kind `deploy_failed` (~458)
- `mcp/src/services/monitor.py` — `_gather_targets()` (~53–94)
- `mcp/src/services/normalizer.py` — `ZapJsonNormalizer` (~667–746), factory (~754–796)
- `mcp/src/services/github_client.py` — `dispatch_workflow()` (~82–95)
- `mcp/src/api/projects.py` — `/projects/{id}/integration` (~303–403)
- `mcp/config/profiles/github-actions-default.yml` — prefix `dast-reports-`
