# Day 23 Lab Reflection

**Student:** Đoàn Minh Quang  
**Student ID (MHV):** 2A202600757  
**Submission date:** 2026-06-29  
**Lab repo URL:** https://github.com/dquangai/2A202600757-DoanMinhQuang-Day23  

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (29.3.1)
Compose v2:    OK  (5.1.0)
RAM available: 3.83 GB (OK)
Ports free:    OK
Report written: D:\Users\quand\2A202600757-DoanMinhQuang-Day23\00-setup\setup-report.json
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+70s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+35s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

I was surprised by how Alertmanager group routing and inhibition rules work. For instance, if the whole service is down (`ServiceDown`), the inhibition rules prevent Alertmanager from spamming warning alerts about latency spikes (`HighInferenceLatency`) on that same service. This deduplication of alert noise is extremely critical in production to prevent SRE alert fatigue.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{"model": "llama3-mock", "input_tokens": 10, "output_tokens": 14, "quality": 0.746, "duration_seconds": 0.2458, "trace_id": "3e20d01a6cb4148ef66de7298ecd3758", "event": "prediction served", "level": "info", "timestamp": "2026-06-29T08:11:42.943369Z"}
```

This log line correlates directly to Jaeger trace `3e20d01a6cb4148ef66de7298ecd3758`.

### Tail-sampling math

Given a service producing `N` traces / second, with:
- $P(\text{error}) = 0.01$ (kept at 100%)
- $P(\text{slow} \land \neg\text{error}) = 0.01$ (kept at 100%)
- $P(\text{healthy}) = 0.98$ (probabilistically sampled at 1%)

The fraction of traces kept by the tail-sampling policy is:

$$
\text{sampled} = N \times (0.01 \times 1.0 + 0.01 \times 1.0 + 0.98 \times 0.01) = N \times 0.0298 \approx 2.98\%
$$

This results in a cost reduction of **97.02%** compared to keeping all traces, while ensuring 100% of errors and tail-latency outliers are captured.

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

### Which test fits which feature?

* **`prompt_length`**: **KS Test (Kolmogorov-Smirnov)** or **PSI**. Continuous numerical feature representing user input characteristics. KS is non-parametric and evaluates if two continuous distributions differ. PSI is perfect for binned numerical monitoring to detect if prompt length distributions shift (e.g., users starting to write longer prompts).
* **`embedding_norm`**: **KS Test**. Since embedding norms are continuous float variables, a non-parametric KS test is ideal for detecting variance/norm shifts without assuming normality.
* **`response_length`**: **PSI**. Suitable for detecting shifting outputs in production bins (e.g. system starting to output extremely short/empty answers).
* **`response_quality`**: **KL Divergence** or **MMD (Maximum Mean Discrepancy)**. Quality scores represent evaluation scores bounded in $[0,1]$. KL divergence measures the information loss between reference quality distribution and current quality distribution. MMD is great for validating quality distributions dynamically without assuming parametric shapes.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

The model-serving metric (`llama.cpp` tokens/second) was the hardest to expose. Unlike python apps that can easily import `prometheus_client` or OpenTelemetry libraries, `llama.cpp` is written in C++ and doesn't natively expose Prometheus endpoints. We had to use a sidecar metrics-collector script that queries `llama.cpp`'s HTTP `/slots` or `/health` endpoints and translates the raw metrics into Prometheus format. This extra hop adds complexity and latency in metrics gathering.

---

## 6. The single change that mattered most

The single change that mattered most was **defining the `inference_quality_score` metric as the "4th pillar" of GenAI SRE observability**. In classic SRE, we monitor the RED/USE metrics (Requests, Errors, Duration / Utilization, Saturation, Errors). However, an LLM service can be 100% healthy at the system level (HTTP 200, latency < 200ms) but output completely gibberish, toxic, or hallucinated responses. By injecting evaluation-as-a-metric into the real-time Prometheus flow, we track the quality of the model dynamically.

This aligns directly with **Lecture Deck §10 (AI-Specific: Drift & Eval)** and **§14 (Loop & Self-Improvement Flywheel)**. Instantly alerting on a drop in the quality score allows SREs to catch silent model regressions and trigger automatic data logging to retrain or fine-tune models before the end-users notice a decline in assistant usefulness.
