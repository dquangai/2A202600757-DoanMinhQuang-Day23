# 📊 Day 23 — Observability Stack Lab Report

**Họ và tên:** Đoàn Minh Quang  
**Mã học viên (MHV):** 2A202600757  
**Ngày nộp:** 2026-06-29  
**Repo:** `2A202600757-DoanMinhQuang-Day23`

---

## 1. Tổng quan

Bài lab Day 23 xây dựng một **full-stack observability platform** cho dịch vụ AI/LLM inference, bao gồm:

| Thành phần | Công cụ | Port |
|---|---|---|
| AI Inference API | FastAPI + Uvicorn | 8000 |
| Metrics | Prometheus | 9090 |
| Alerting | Alertmanager | 9093 |
| Dashboards | Grafana | 3000 |
| Logging | Loki | 3100 |
| Tracing | Jaeger | 16686 |
| Collector | OTel Collector | 4317/4318 |

---

## 2. Kết quả kiểm tra tự động (verify.py)

```
[PASS] 00-setup: setup-report.json committed
[PASS] 01: app /healthz reachable
[PASS] 01: /metrics exposes inference_requests_total
[PASS] 02: Prometheus reachable
[PASS] 02: Grafana reachable
[PASS] 02: Alertmanager reachable
[PASS] 02: 3 Day-23 dashboards loaded (found=4)
[PASS] 03: Jaeger UI reachable
[PASS] 03: Loki ready
[PASS] 03: OTel Collector self-metrics reachable
[PASS] 04: drift-summary.json shows at least one drifted feature
[PASS] submission: REFLECTION.md exists and is non-trivial

Result: 12/12 checks passed ✅
```

---

## 3. Screenshots

### 3.1 Grafana — AI Service Overview Dashboard

Dashboard tổng quan hiển thị 6 panel chính: Request Rate, Error Rate, Latency P95, Active Requests, Quality Score, và Token Usage.

![AI Service Overview Dashboard](submission/screenshots/dashboard-overview.png)

---

### 3.2 Grafana — SLO Burn Rate Dashboard

Dashboard SLO burn-rate theo multi-window strategy (1h, 6h, 3d) để phát hiện vi phạm error budget sớm.

![SLO Burn Rate Dashboard](submission/screenshots/slo-burn-rate.png)

---

### 3.3 Grafana — Cost & Tokens Dashboard

Dashboard theo dõi chi phí token sử dụng, tổng input/output tokens, và estimated cost per request.

![Cost & Tokens Dashboard](submission/screenshots/cost-tokens.png)

---

### 3.4 Grafana — Cross-Day Stack (Integrative) Dashboard

Dashboard tích hợp liên ngày, hiển thị metrics từ Day 16–22. Day 19 (Qdrant Collections = 3) và Day 20 (llama.cpp tokens/sec ≈ 20) đã kết nối thành công qua stub metrics.

![Cross-Day Stack Dashboard](submission/screenshots/dashboard-crossday.png)

---

### 3.5 Alertmanager UI

Giao diện Alertmanager hiển thị trạng thái alert groups và routing configuration.

![Alertmanager UI](submission/screenshots/alertmanager-firing.png)

---

### 3.6 Mock Slack — Alert Firing

Mock Slack Webhook Server nhận alert `ServiceDown` khi container `day23-app` bị dừng. Alert fired sau 70 giây.

![Slack Alert Firing](submission/screenshots/slack-firing.png)

---

### 3.7 Mock Slack — Alert Resolved

Sau khi restart container, alert tự động resolved sau 35 giây.

![Slack Alert Resolved](submission/screenshots/slack-resolved.png)

---

### 3.8 Jaeger — Trace Visualization

Trace của request `POST /predict` hiển thị 3 child spans: `embed-text`, `vector-search`, `generate-tokens`.

![Jaeger Trace](submission/screenshots/jaeger-trace.png)

---

### 3.9 Jaeger — GenAI Semantic Attributes

Span `generate-tokens` hiển thị các GenAI semantic convention attributes: `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reason`.

![Jaeger GenAI Attributes](submission/screenshots/jaeger-attrs.png)

---

### 3.10 Evidently — Drift Detection Report

Báo cáo drift detection phát hiện 2/4 features bị drift: `prompt_length` (PSI=3.461) và `response_quality` (PSI=8.849).

![Evidently Drift Report](submission/screenshots/drift-report.png)

---

## 4. Kết quả Drift Detection

```json
{
  "prompt_length":    { "psi": 3.461,  "drift": "yes" },
  "embedding_norm":   { "psi": 0.019,  "drift": "no"  },
  "response_length":  { "psi": 0.016,  "drift": "no"  },
  "response_quality": { "psi": 8.849,  "drift": "yes" }
}
```

---

## 5. Alert Drill Timeline

| Thời điểm | Sự kiện | Kết quả |
|---|---|---|
| T₀ | `docker stop day23-app` | Container dừng |
| T₀ + 70s | `ServiceDown` fired | Alertmanager gửi webhook → Mock Slack |
| T₁ | `docker start day23-app` | Container khởi động lại |
| T₁ + 35s | Alert resolved | Alertmanager gửi resolved webhook |

---

## 6. Structured Log — Trace Correlation

```json
{
  "model": "llama3-mock",
  "input_tokens": 10,
  "output_tokens": 14,
  "quality": 0.746,
  "duration_seconds": 0.2458,
  "trace_id": "3e20d01a6cb4148ef66de7298ecd3758",
  "event": "prediction served",
  "level": "info",
  "timestamp": "2026-06-29T08:11:42.943369Z"
}
```

→ Trace ID `3e20d01a6cb4148ef66de7298ecd3758` khớp với trace trong Jaeger UI.

---

## 7. Kết luận

Bài lab đã hoàn thành đầy đủ **100 điểm** theo rubric:

| Track | Điểm | Mô tả |
|---|---|---|
| 00 Setup | ✅ 5 | `setup-report.json` generated |
| 01 Instrument | ✅ 15 | 5 metrics exposed on `/metrics` |
| 02 Dashboards | ✅ 20 | 4 dashboards loaded, load test completed |
| 02 Alerts | ✅ 15 | Alert drill: fire + resolve via mock Slack |
| 03 Tracing | ✅ 15 | 3-span trace, log-trace correlation, tail-sampling |
| 04 Drift | ✅ 15 | PSI/KL/KS computed, Evidently HTML report |
| 05 Integration | ✅ 15 | Day 19 + Day 20 stubs scraped by Prometheus |
| **Tổng** | **100** | |

---

*Báo cáo được tạo bởi Đoàn Minh Quang — MHV: 2A202600757*
