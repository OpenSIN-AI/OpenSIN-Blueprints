# 🏘️ BLUEPRINT MODUL 02: 17-ROOM DISTRIBUTED FORTRESS ARCHITECTURE
## Version 2.0 (2026-01-26) | Elite Status: 500+ Lines | Authority: Sisyphus CEO

---

## TABLE OF CONTENTS
1. Architecture Overview & Philosophy
2. 17-Zimmer Complete Specification Matrix
3. Inter-Zimmer Communication Protocol
4. Data Flow Architecture (Screenshot → Success)
5. Resilience & Failover Topology
6. Deployment Topologies (Development, Staging, Production)
7. Security Architecture & Isolation
8. Performance Characteristics & SLAs
9. Health Check Protocol & Monitoring
10. Authority & Version Control

---

## 1. ARCHITECTURE OVERVIEW & PHILOSOPHY

### Core Principles

**Principle 1: Modular Sovereignty**
Each zimmer is completely isolated in its own Docker container with:
- Independent memory space (no shared state)
- Isolated network (internal 172.20.0.x bridge network)
- Dedicated volumes (persistence per zimmer)
- Clear responsibility (single function, do it well)

**Principle 2: Async-First Parallelism**
No blocking operations. All I/O is asynchronous:
- FastAPI with Uvicorn workers (async request handling)
- asyncio.gather() for parallel model inference
- Redis pub/sub for inter-zimmer messaging (not HTTP blocking)
- Timeout protection on all external calls

**Principle 3: Fault Tolerance by Design**
Single points of failure eliminated:
- Multiple API providers (Gemini, Mistral, CapMonster, OpenAI fallback)
- Regional distribution (EU, US, APAC — each has full stack)
- Circuit breakers (auto-disable failing API for 5min)
- Dead letter queues (DLQ) for failed requests

**Principle 4: Observability First**
Every action is logged, traced, and metrically captured:
- Prometheus metrics on every decision point
- Structured logging (JSON, searchable)
- Distributed tracing (request UUID across all zimmers)
- Real-time dashboards (Grafana)

---

## 2. 17-ZIMMER COMPLETE SPECIFICATION MATRIX

### Zimmer-01: n8n Orchestrator (Workflow Automation)
```
ROLE: Workflow automation, rule engine, decision orchestration
IMAGE: n8n:latest (or n8n:arm64 for M1)
INTERNAL IP: 172.20.0.10
PORT: 5678
CPU/RAM: 2 cores / 2GB (M1-native)
PERSISTENCE: Volume /n8n-data → docker-compose

HEALTH CHECK:
  - HTTP GET http://172.20.0.10:5678/healthz (expect 200)
  - Interval: 30s, Timeout: 5s, Retries: 3

INTER-ZIMMER PROTOCOL:
  - Listens on HTTP POST /webhook/execute
  - Accepts: {workflow_id, payload, timeout_ms}
  - Returns: {status, result, duration_ms}
  - Timeout: 30s

DEPENDENCIES:
  - Postgres (Zimmer-10) for workflow storage
  - Redis (implicit, for caching)

EXAMPLE REQUEST:
  POST http://172.20.0.10:5678/webhook/execute
  {
    "workflow_id": "captcha_consensus_workflow",
    "payload": {"screenshot": "base64...", "timeout_ms": 20000},
    "timeout_ms": 25000
  }

FAILURE MODES:
  - Workflow not found → return {status: "error", code: "WORKFLOW_NOT_FOUND"}
  - Timeout → auto-return after timeout_ms with partial results
  - Postgres unavailable → queue in Redis, retry every 5s
```

---

### Zimmer-02: Chronos-Stratege (Strategic Timing Engine)
```
ROLE: Behavioral timing randomization, anti-detection timing
IMAGE: custom/chronos-stratege:v2.0
INTERNAL IP: 172.20.0.2
PORT: 3001
CPU/RAM: 1 core / 512MB (M1-native)
PERSISTENCE: Volume /chronos-data → request patterns, timing curves

HEALTH CHECK:
  - HTTP GET http://172.20.0.2:3001/health (expect 200 OK)
  - Interval: 30s, Timeout: 5s, Retries: 3

INTER-ZIMMER PROTOCOL:
  - HTTP POST /timing/generate
  - Input: {action_type, target_latency_ms, randomness_level}
  - Output: {delays: [ms], total_duration_ms, bézier_curve}
  - Timeout: 2s (must be fast, used in request path)

EXAMPLE REQUEST:
  POST http://172.20.0.2:3001/timing/generate
  {
    "action_type": "mouse_movement",
    "target_latency_ms": 150,
    "randomness_level": "high"
  }
  Response: {
    "delays": [23, 45, 67, 15],
    "total_duration_ms": 150,
    "bézier_curve": "quadratic"
  }

FAILURE MODES:
  - Service down → fallback to fixed timing (200ms)
  - Invalid action_type → return error code, caller handles fallback
```

---

### Zimmer-03: Agent Zero (Code Generation & Consensus Engine)
```
ROLE: 5-Model parallel consensus orchestration, tier-based decision making
IMAGE: custom/agent-zero:v3.0
INTERNAL IP: 172.20.0.50
PORT: 8000
CPU/RAM: 4 cores / 4GB (M1-native, GPU if available)
PERSISTENCE: Volume /agent-zero-cache → model caches, inference results

HEALTH CHECK:
  - HTTP GET http://172.20.0.50:8000/health (expect 200)
  - Deep health: GET /health/detailed (checks all 5 model APIs)
  - Interval: 10s, Timeout: 10s, Retries: 5

INTER-ZIMMER PROTOCOL (Primary Interface):
  - HTTP POST /consensus/solve
  - Input: {screenshot, timeout_ms, preferred_models}
  - Output: {result, confidence, model_votes, latency_ms, cost_usd}
  - Timeout: 25s (configurable)

CONSENSUS LOGIC:
  Tier 1 (Majority):
    - Run: Gemini, Mistral, YOLO (3 parallel)
    - Decision: 2/3 agreement → return immediately
    - Confidence threshold: > 0.9
    
  Tier 2 (Dual):
    - Run: Gemini + Mistral if Tier 1 incomplete
    - Decision: 2/2 agreement with 0.9+ confidence → return at 8s mark
    
  Tier 3 (CapMonster):
    - Run: CapMonster (hCaptcha specialist)
    - Decision: Use if Tier 1 + 2 fail, or hCaptcha type detected
    
  Tier 4 (Escalation):
    - Run: Human fallback or mark as failed
    - Decision: If all tiers < 0.6 confidence

EXAMPLE REQUEST:
  POST http://172.20.0.50:8000/consensus/solve
  {
    "screenshot": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADU...",
    "timeout_ms": 20000,
    "preferred_models": ["gemini", "mistral", "yolo"]
  }

EXAMPLE RESPONSE:
  {
    "result": "2, 4, 6, 8",
    "confidence": 0.94,
    "model_votes": {
      "gemini": {answer: "2, 4, 6, 8", confidence: 0.96},
      "mistral": {answer: "2, 4, 6, 8", confidence: 0.92},
      "yolo": {answer: "2, 4, 6, 8", confidence: 0.92}
    },
    "latency_ms": 5200,
    "cost_usd": 0.008,
    "solver_used": "tier1_consensus"
  }

FAILURE MODES:
  - Service down → circuit breaker, return error to client
  - Model API down → skip to next tier, retry remaining models
  - Timeout → return best available result before timeout
```

---

### Zimmer-04: Opencode-Sekretaer (Configuration & Secret Management)
```
ROLE: API key management, configuration server, environment secrets
IMAGE: vault:latest (or hashicorp/vault)
INTERNAL IP: 172.20.0.4
PORT: 8200
CPU/RAM: 1 core / 512MB (M1-native)
PERSISTENCE: Volume /vault-data → encrypted secrets, audit logs

HEALTH CHECK:
  - HTTP GET http://172.20.0.4:8200/v1/sys/health (expect 200)
  - Interval: 60s, Timeout: 5s, Retries: 3

INTER-ZIMMER PROTOCOL:
  - HTTP POST /v1/secret/data/api-keys/{provider}
  - Authentication: Bearer token (Vault root token in env)
  - Input: {key_id}
  - Output: {api_key, region, quota_remaining}
  - Timeout: 3s

STORED SECRETS:
  - gemini_api_key
  - mistral_api_key
  - capmonster_api_key
  - openai_api_key (fallback)
  - stripe_api_key (billing)
  - slack_webhook (alerts)
  - postgres_password
  - redis_password

EXAMPLE REQUEST:
  POST http://172.20.0.4:8200/v1/secret/data/api-keys/gemini
  Headers: {Authorization: "Bearer s.xxxxxx"}
  Response: {
    "api_key": "AIzaSyDxxx...",
    "region": "us-central1",
    "quota_remaining": 5000,
    "next_reset": "2026-01-27T00:00:00Z"
  }

FAILURE MODES:
  - Service down → in-memory cache (20min TTL) returns last known secrets
  - Secret expired → circuit breaker, disable API until refreshed
  - Auth failed → return error, caller handles graceful degradation
```

---

### Zimmer-05: Steel Stealth (Proxy & TLS Fingerprint Engine)
```
ROLE: HTTP request proxying, TLS fingerprint randomization, header spoofing
IMAGE: custom/steel-stealth:v2.0
INTERNAL IP: 172.20.0.20
PORT: 3000
CPU/RAM: 2 cores / 1GB (M1-native)
PERSISTENCE: Volume /stealth-data → proxy rotation list, TLS curves database

HEALTH CHECK:
  - HTTP GET http://172.20.0.20:3000/health (expect 200)
  - Interval: 30s, Timeout: 5s, Retries: 3

INTER-ZIMMER PROTOCOL:
  - HTTP POST /proxy/execute
  - Input: {url, method, headers, body, tls_config}
  - Output: {status_code, headers, body, latency_ms}
  - Timeout: 30s

EXAMPLE REQUEST:
  POST http://172.20.0.20:3000/proxy/execute
  {
    "url": "https://captcha.example.com/solve",
    "method": "POST",
    "headers": {"User-Agent": "Mozilla/5.0...", "Referer": "..."},
    "body": "{\"solution\": \"...\"}",
    "tls_config": {
      "cipher_suites": ["TLS_AES_256_GCM_SHA384", "TLS_CHACHA20_POLY1305_SHA256"],
      "supported_curves": ["secp256r1", "x25519"],
      "extensions_order": "random"
    }
  }

EVASION FEATURES:
  - JA3 fingerprint randomization (cipher suite rotation)
  - TLS extension order randomization
  - User-Agent rotation (50+ real browsers)
  - Accept-Language matching (geo-matched)
  - Custom header injection (DNT, Referer consistency)
  - Proxy rotation (up to 100 proxies, round-robin)

FAILURE MODES:
  - All proxies exhausted → fallback to direct connection
  - TLS negotiation fails → retry with different cipher suites
  - Request timeout → return 504 Gateway Timeout
```

---

### Zimmer-06: Skyvern Auge (Vision Analysis - Skyvern)
```
ROLE: Visual element detection, interactive element identification
IMAGE: skyvern/skyvern:latest (or community build for M1)
INTERNAL IP: 172.20.0.30
PORT: 8000
CPU/RAM: 2 cores / 2GB (M1-native)
PERSISTENCE: Volume /skyvern-data → element cache, visual analysis results

HEALTH CHECK:
  - HTTP GET http://172.20.0.30:8000/health (expect 200)
  - Interval: 30s, Timeout: 5s, Retries: 3

INTER-ZIMMER PROTOCOL:
  - HTTP POST /analyze
  - Input: {screenshot, task_description, timeout_ms}
  - Output: {elements: [{selector, type, text, bbox}], actions: [...]}
  - Timeout: 15s

EXAMPLE REQUEST:
  POST http://172.20.0.30:8000/analyze
  {
    "screenshot": "base64...",
    "task_description": "Find and click the 'submit' button",
    "timeout_ms": 10000
  }

EXAMPLE RESPONSE:
  {
    "elements": [
      {
        "selector": "button#submit",
        "type": "button",
        "text": "Submit",
        "bbox": {x: 100, y: 200, w: 80, h: 40},
        "confidence": 0.98
      },
      {
        "selector": "input[type=text]",
        "type": "input",
        "text": "",
        "bbox": {x: 100, y: 100, w: 200, h: 30},
        "confidence": 0.95
      }
    ],
    "actions": [
      {action: "click", selector: "button#submit", confidence: 0.98}
    ]
  }

FAILURE MODES:
  - Image decode error → return 400 Bad Request
  - Task unsupported → return 422 Unprocessable Entity with reason
  - Timeout → return best partial results before deadline
```

---

### Zimmer-07: Stagehand Detektiv (Intelligence & Learning Loop)
```
ROLE: Forensic analysis, differential feedback, model retraining signals
IMAGE: custom/stagehand-detektiv:v2.0
INTERNAL IP: 172.20.0.7
PORT: 3000
CPU/RAM: 2 cores / 2GB (M1-native)
PERSISTENCE: Volume /zimmer07-data → forensic database, learning signals

HEALTH CHECK:
  - HTTP GET http://172.20.0.7:3000/health (expect 200)
  - Interval: 60s, Timeout: 10s, Retries: 3

INTER-ZIMMER PROTOCOL:
  - HTTP POST /forensics/analyze
  - Input: {request_id, screenshot, expected_result, actual_result, models_used}
  - Output: {root_cause, recommendation, learning_signal}
  - Timeout: 5s (async processing)

FORENSIC ANALYSIS:
  - Model accuracy per type (reCAPTCHA v2 vs v3 vs hCaptcha)
  - Detection vector identification (new blocker patterns)
  - Evasion effectiveness (TLS fingerprint randomization working?)
  - Consensus failure rates (identify weak model pairs)

EXAMPLE REQUEST:
  POST http://172.20.0.7:3000/forensics/analyze
  {
    "request_id": "req_123456",
    "screenshot": "base64...",
    "expected_result": "2, 4, 6, 8",
    "actual_result": "2, 4, 7, 8",
    "models_used": ["gemini", "mistral", "yolo"],
    "detection_triggered": false,
    "latency_ms": 4800
  }

RESPONSE:
  {
    "root_cause": "YOLO confidence 0.71 (threshold 0.75) — borderline grid detection",
    "recommendation": "Investigate YOLO training data for multi-select grids",
    "learning_signal": {
      "model": "yolo",
      "issue_type": "BORDERLINE_CONFIDENCE",
      "captcha_type": "recaptcha_v2",
      "priority": "medium"
    }
  }

FAILURE MODES:
  - Database connection fails → queue in Redis, batch process later
  - Analysis timeout → return empty recommendation, log for manual review
```

---

### Zimmer-08: QA-Prüfer (Quality Assurance & Testing)
```
ROLE: Automated testing, regression detection, solve rate verification
IMAGE: custom/qa-prufer:v2.0
INTERNAL IP: 172.20.0.8
PORT: 8080
CPU/RAM: 2 cores / 1GB (M1-native)
PERSISTENCE: Volume /qa-data → test results, regression tracking

HEALTH CHECK:
  - HTTP GET http://172.20.0.8:8080/health (expect 200)
  - Interval: 3600s (1 hour, low frequency)

TEST SUITE:
  - Unit tests (model inference, consensus logic)
  - Integration tests (end-to-end solve)
  - Regression tests (solve rate tracking per CAPTCHA type)
  - Performance tests (latency p50/p95/p99)
  - Security tests (detection rate, evasion effectiveness)

INTER-ZIMMER PROTOCOL:
  - HTTP POST /test/run
  - Input: {test_suite, captcha_samples_count}
  - Output: {passed, failed, metrics}
  - Timeout: 300s (5 minutes for full suite)

EXAMPLE REQUEST:
  POST http://172.20.0.8:8080/test/run
  {
    "test_suite": "regression",
    "captcha_samples_count": 100
  }

RESPONSE:
  {
    "passed": 98,
    "failed": 2,
    "solve_rate": 0.98,
    "latency": {p50: 5200, p95: 12100, p99: 18900},
    "detection_rate": 0.01,
    "metrics": {
      "recaptcha_v2": {solve_rate: 0.99, latency_p50: 4800},
      "recaptcha_v3": {solve_rate: 0.96, latency_p50: 8200},
      "hcaptcha": {solve_rate: 0.98, latency_p50: 6100}
    }
  }

FAILURE MODES:
  - Test sample failure → log, investigate, report
  - Timeout → abort test, report partial results
  - Regression detected → trigger PagerDuty SEV1 alert
```

---

### Zimmer-09: Clawdbot-Bote (API Router & Load Balancing)
```
ROLE: HTTP request routing, load balancing, rate limiting
IMAGE: nginx:latest or traefik:latest
INTERNAL IP: 172.20.0.9
PORT: 8080
CPU/RAM: 1 core / 512MB (M1-native)
PERSISTENCE: None (stateless)

HEALTH CHECK:
  - HTTP GET http://172.20.0.9:8080/health (expect 200)
  - Interval: 10s, Timeout: 5s, Retries: 3

ROUTING RULES:
  - /api/solve → Agent Zero (Zimmer-03)
  - /api/analyze → Skyvern (Zimmer-06)
  - /api/timing → Chronos (Zimmer-02)
  - /api/proxy → Steel Stealth (Zimmer-05)
  - /health → Pass-through to all zimmers
  - /metrics → Prometheus exporter (Zimmer-17)

LOAD BALANCING:
  - Round-robin across all instances of Zimmer-03 (if scaled)
  - Circuit breaker (disable unhealthy upstream for 30s)
  - Retry logic (3 retries on 5xx errors)
  - Request timeout: 30s

RATE LIMITING:
  - Per-IP: 100 requests/minute
  - Per-API-key: 10,000 requests/day
  - Burst limit: 20 requests/second (token bucket)

EXAMPLE REQUEST (pass-through):
  POST http://172.20.0.9:8080/api/solve
  {screenshot: "...", timeout_ms: 20000}
  
  Routes to: http://172.20.0.50:8000/consensus/solve
  Returns: Same response format (transparent proxy)

FAILURE MODES:
  - All upstreams unavailable → return 503 Service Unavailable
  - Rate limit exceeded → return 429 Too Many Requests
  - Invalid request → return 400 Bad Request
```

---

### Zimmer-10: Postgres Bibliothek (Primary Database)
```
ROLE: Persistent data storage (users, sessions, billing, audit logs)
IMAGE: postgres:15-alpine (or arm64 specific)
INTERNAL IP: 172.20.0.10
PORT: 5432
CPU/RAM: 2 cores / 2GB (M1-native, SSD preferred)
PERSISTENCE: Volume /postgres-data → critical (backup daily)

HEALTH CHECK:
  - Command: pg_isready -h 172.20.0.10 -U postgres
  - Interval: 30s, Timeout: 5s, Retries: 3

SCHEMAS:
  - users (id, email, api_key, tier, created_at)
  - sessions (id, user_id, auth_token, expires_at)
  - solves (id, user_id, screenshot, result, latency_ms, cost_usd, timestamp)
  - audit_log (id, action, actor_id, resource, change, timestamp)

INTER-ZIMMER PROTOCOL:
  - Direct TCP connection (not HTTP)
  - Connection pooling via PgBouncer (Zimmer-10 adjacent)
  - Max connections: 100 (tune per load)

EXAMPLE QUERY:
  SELECT COUNT(*) FROM solves 
  WHERE created_at > NOW() - INTERVAL '1 hour' 
  AND user_id = 'user_123'

BACKUP STRATEGY:
  - Daily full backup (pg_dump → S3)
  - Point-in-time recovery (WAL archiving → S3)
  - Restore test weekly (verify recoverability)
  - Retention: 30 days

FAILURE MODES:
  - Connection refused → circuit breaker, fallback to read-only cache
  - Disk full → trigger PagerDuty SEV1 (critical)
  - Slow queries → log, optimize indexes, notify CTO
```

---

### Zimmer-11: Dashboard Zentrale (Monitoring & Observability)
```
ROLE: Real-time KPI dashboard, Grafana visualizations
IMAGE: grafana:latest
INTERNAL IP: 172.20.0.60
PORT: 3000
CPU/RAM: 1 core / 1GB (M1-native)
PERSISTENCE: Volume /grafana-data → dashboards, datasources

HEALTH CHECK:
  - HTTP GET http://172.20.0.60:3000/api/health (expect 200)
  - Interval: 60s, Timeout: 5s, Retries: 3

DASHBOARDS:
  - Executive KPI (revenue, margin, cost per solve)
  - Technical Health (solve rate, latency, detection rate)
  - Forensic (failed consensus, escalations, error logs)

EXAMPLE METRICS (from Prometheus):
  - sin_solver_requests_total{model="gemini"}
  - sin_solver_latency_ms{quantile="p50|p95|p99"}
  - sin_solver_consensus_confidence
  - sin_solver_detection_triggered{blocker_type="cloudflare"}

DATASOURCES:
  - Prometheus (172.20.0.100:9090)
  - Postgres (172.20.0.10:5432)

FAILURE MODES:
  - Service down → dashboards unavailable (but metrics still collected in Prometheus)
  - Slow queries → increase timeout, add indexes to underlying data
```

---

### Zimmer-12: Evolution Optimizer (Performance Tuning Agent)
```
ROLE: Continuous optimization (thresholds, timeouts, model weights)
IMAGE: custom/evolution-optimizer:v2.0
INTERNAL IP: 172.20.0.12
PORT: 8080
CPU/RAM: 1 core / 512MB (M1-native)
PERSISTENCE: Volume /optimizer-data → tuning parameters, A/B test results

HEALTH CHECK:
  - HTTP GET http://172.20.0.12:8080/health (expect 200)
  - Interval: 300s (5 minutes, low frequency)

OPTIMIZATION LOOPS:
  - Confidence thresholds (when to return immediately vs. wait for consensus)
  - Timeout tuning (how long to wait for each tier)
  - Model weighting (adjust consensus voting weights)
  - Cost optimization (shift to cheaper models if similar quality)

EXAMPLE OPTIMIZATION:
  Current: YOLO confidence 0.92 returns immediately
  A/B Test: Group A (0.92), Group B (0.94)
  Result: Group B has 0.2% lower solve rate, 18% lower cost
  Decision: Update threshold to 0.94

INTER-ZIMMER PROTOCOL:
  - HTTP POST /optimize/tuning-parameters
  - Input: {metric_name, target_value, constraint}
  - Output: {new_parameters, expected_impact, rollback_plan}
  - Timeout: 60s

FAILURE MODES:
  - Optimization causes regression → auto-rollback, notify CTO
  - Service down → use cached parameters (48h TTL)
```

---

### Zimmer-13: API Brain (Vault & Key Rotation)
```
ROLE: API quota management, key rotation, provider health monitoring
IMAGE: custom/api-brain:v2.0
INTERNAL IP: 172.20.0.31
PORT: 8000
CPU/RAM: 1 core / 512MB (M1-native)
PERSISTENCE: Volume /api-brain-data → quota tracking, rotation history

HEALTH CHECK:
  - HTTP GET http://172.20.0.31:8000/health (expect 200)
  - Interval: 60s, Timeout: 5s, Retries: 3

QUOTA MANAGEMENT:
  - Gemini: 60,000 requests/day (track hourly usage)
  - Mistral: 100,000 requests/day
  - CapMonster: $100/day budget (track spend)
  - Fallback routing if quota nearing

INTER-ZIMMER PROTOCOL:
  - HTTP POST /quota/check
  - Input: {provider}
  - Output: {quota_remaining, quota_reset_time, available}
  - Timeout: 2s

EXAMPLE REQUEST:
  POST http://172.20.0.31:8000/quota/check
  {"provider": "gemini"}
  
  Response: {
    "quota_remaining": 5000,
    "quota_total": 60000,
    "quota_reset_time": "2026-01-27T00:00:00Z",
    "available": true,
    "estimated_time_to_exhaustion_hours": 2.5
  }

FAILURE MODES:
  - Quota exhausted → return unavailable, trigger fallback routing
  - API unreachable → use cached quota (15min TTL), assume available
  - Key rotation → coordinated with Zimmer-04 (Vault)
```

---

### Zimmer-14: Worker Arbeiter (Background Job Queue)
```
ROLE: Async job processing (batch solves, model retraining, cleanup)
IMAGE: custom/worker-arbeiter:v2.0
INTERNAL IP: 172.20.0.14
PORT: 8080
CPU/RAM: 2 cores / 1GB (M1-native)
PERSISTENCE: Volume /worker-data → job queue, processing logs

HEALTH CHECK:
  - HTTP GET http://172.20.0.14:8080/health (expect 200)
  - Interval: 60s, Timeout: 5s, Retries: 3

JOB TYPES:
  - batch_solve (process 1000 CAPTCHAs, save results)
  - cleanup_cache (old entries > 7 days)
  - audit_log_archive (daily archival to cold storage)
  - model_retraining (run differential feedback on failures)

QUEUE: Redis (172.20.0.15 - shared across all workers)
  - Job timeout: 30min per job
  - Retry policy: 3 retries on failure, exponential backoff
  - Dead letter queue: jobs that fail 3x → manual review

EXAMPLE JOB:
  {
    "job_id": "job_987654",
    "type": "batch_solve",
    "payload": {
      "captcha_count": 1000,
      "types": ["recaptcha_v2", "hcaptcha"],
      "save_results": true
    },
    "created_at": "2026-01-26T22:00:00Z",
    "expires_at": "2026-01-26T22:30:00Z"
  }

FAILURE MODES:
  - Job timeout → retry with exponential backoff
  - Dependency unavailable → queue job, retry every 5min
  - Worker crash → Redis maintains queue, another worker picks up job
```

---

### Zimmer-15: Surfsense Archiv (Vector Database & Semantic Cache)
```
ROLE: CAPTCHA embedding storage, semantic similarity search, caching
IMAGE: qdrant/qdrant:latest
INTERNAL IP: 172.20.0.15
PORT: 6333
CPU/RAM: 2 cores / 2GB (M1-native, SSD preferred)
PERSISTENCE: Volume /qdrant-data → embeddings, index

HEALTH CHECK:
  - HTTP GET http://172.20.0.15:6333/health (expect 200)
  - Interval: 60s, Timeout: 5s, Retries: 3

COLLECTIONS:
  - captcha_embeddings (vectors: 384-dim, metadata: captcha_type, solve_result)
  - solution_cache (vectors: solution embeddings, TTL: 7 days)

INTER-ZIMMER PROTOCOL:
  - HTTP POST /collections/captcha_embeddings/points/search
  - Input: {vector, limit, score_threshold}
  - Output: [{id, score, payload}]
  - Timeout: 5s

EXAMPLE REQUEST:
  POST http://172.20.0.15:6333/collections/captcha_embeddings/points/search
  {
    "vector": [0.123, 0.456, ...],  // embedding of current screenshot
    "limit": 10,
    "score_threshold": 0.8
  }

  Response: [
    {
      "id": "captcha_12345",
      "score": 0.95,
      "payload": {
        "type": "recaptcha_v2",
        "result": "2, 4, 6, 8",
        "confidence": 0.98,
        "cached_at": "2026-01-26T20:00:00Z"
      }
    }
  ]

CACHE HIT BENEFIT:
  - If embedding similarity > 0.90: return cached solution (0% API cost, 10ms latency)
  - Saves API calls for repeated CAPTCHAs

FAILURE MODES:
  - Service down → return empty search results (no cache hit penalty)
  - High disk usage → implement retention policy (7-day TTL)
  - Slow search → add more replicas, tune search parameters
```

---

### Zimmer-16: Supabase Zimmer (Real-time Analytics DB)
```
ROLE: Real-time analytics, WebSocket subscriptions, user data
IMAGE: supabase/supabase:latest (or postgres + realtime)
INTERNAL IP: 172.20.0.16
PORT: 5432 (Postgres), 3000 (GraphQL)
CPU/RAM: 2 cores / 2GB (M1-native)
PERSISTENCE: Volume /supabase-data → user profiles, billing data

HEALTH CHECK:
  - HTTP GET http://172.20.0.16:3000/health (expect 200)
  - Interval: 60s, Timeout: 5s, Retries: 3

TABLES:
  - users (id, email, api_key, subscription_tier, created_at)
  - api_usage (user_id, solves_today, cost_today, timestamp)
  - billing (user_id, monthly_cost, next_billing_date)

REAL-TIME SUBSCRIPTIONS:
  - WebSocket: Listen to user quota changes (notify client when quota nearing)
  - Example: "SELECT * FROM api_usage WHERE user_id = 'user_123'"

FAILURE MODES:
  - Service down → fallback to Zimmer-10 (Postgres direct)
  - Real-time unavailable → revert to polling (10s interval)
```

---

### Zimmer-17: SIN-Plugins (MCP Integration)
```
ROLE: Model Context Protocol (MCP) integration, external tool bridging
IMAGE: custom/sin-plugins-mcp:v2.0
INTERNAL IP: 172.20.0.40
PORT: 8000
CPU/RAM: 1 core / 512MB (M1-native)
PERSISTENCE: Volume /plugins-data → registered plugins, call logs

HEALTH CHECK:
  - HTTP GET http://172.20.0.40:8000/health (expect 200)
  - Interval: 60s, Timeout: 5s, Retries: 3

REGISTERED PLUGINS:
  - playwright (browser automation)
  - git-master (version control)
  - dev-browser (web scraping)
  - frontend-ui-ux (UI insights)

INTER-ZIMMER PROTOCOL:
  - HTTP POST /plugins/execute
  - Input: {plugin_name, function_name, args}
  - Output: {result, duration_ms}
  - Timeout: 60s

EXAMPLE REQUEST:
  POST http://172.20.0.40:8000/plugins/execute
  {
    "plugin_name": "playwright",
    "function_name": "screenshot",
    "args": {"url": "https://captcha.example.com", "wait_ms": 3000}
  }

  Response: {
    "result": "iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADU...",
    "duration_ms": 3500
  }

FAILURE MODES:
  - Plugin not found → return 404 error
  - Timeout → kill process, return timeout error
  - Invalid args → return 400 Bad Request with validation error
```

---

## 3. INTER-ZIMMER COMMUNICATION PROTOCOL

### Request/Response Format

**Standard Request Header**:
```
POST http://[zimmer-ip]:[port]/endpoint
Headers:
  - Content-Type: application/json
  - X-Request-ID: req_123456 (traceable across all zimmers)
  - X-Timeout-Ms: 20000 (caller's timeout, in milliseconds)
  - X-Retry-Count: 0 (incremented on retry)
Authorization: Bearer [api-key] (if required)
```

**Standard Response Format**:
```json
{
  "success": true,
  "data": {...},
  "error": null,
  "timestamp": "2026-01-26T22:47:00Z",
  "duration_ms": 1234,
  "request_id": "req_123456"
}

// On error:
{
  "success": false,
  "data": null,
  "error": {
    "code": "MODEL_TIMEOUT",
    "message": "Gemini inference exceeded 8s timeout",
    "retry_after_ms": 5000
  },
  "timestamp": "2026-01-26T22:47:00Z",
  "request_id": "req_123456"
}
```

### Latency SLAs Between Zimmers

| Source → Dest | Operation | Target Latency | p99 Latency |
|---------------|-----------|----------------|------------|
| Clawdbot (09) → Agent Zero (03) | /consensus/solve | < 100ms | < 200ms |
| Agent Zero (03) → Skyvern (06) | /analyze | < 500ms | < 1000ms |
| Agent Zero (03) → Steel Stealth (05) | /proxy/execute | < 100ms | < 500ms |
| Agent Zero (03) → Chronos (02) | /timing/generate | < 50ms | < 100ms |
| Agent Zero (03) → Vault (04) | /secret/data | < 100ms | < 200ms (cached) |
| Stagehand (07) → Postgres (10) | INSERT forensic log | < 200ms | < 500ms |
| Dashboard (11) → Prometheus | Query metrics | < 500ms | < 2000ms |

### Circuit Breaker Strategy

**Circuit States**:
1. **CLOSED**: Normal operation, requests passing through
2. **OPEN**: Service failing, requests immediately rejected (fail-fast)
3. **HALF_OPEN**: Test if service recovered, allow 1 request

**Trigger Rules**:
- Transition CLOSED → OPEN: 5 consecutive failures OR error rate > 50% for 1 minute
- Stay OPEN: Duration 30 seconds
- Transition OPEN → HALF_OPEN: 30s elapsed, allow 1 test request
- Transition HALF_OPEN → CLOSED: Test request succeeds

**Example**:
```
Agent Zero trying to call Gemini API (via Zimmer-04 Vault)
- Request 1: Success
- Request 2: Timeout
- Request 3: Timeout
- Request 4: Timeout
- Request 5: Timeout (5th failure)
- Circuit breaker OPENS
- Requests 6-100: Immediately rejected with 503 (fail-fast)
- After 30s: Circuit HALF_OPEN
- Request 101: Allowed (test request)
  - If succeeds: Circuit CLOSED, resume normal operation
  - If fails: Circuit OPEN again for another 30s
```

---

## 4. DATA FLOW ARCHITECTURE (Screenshot → Success)

### End-to-End Request Flow

```
CLIENT REQUEST
    ↓
[Zimmer-09: Clawdbot-Bote - Load Balancer]
    ↓ (route /api/solve)
[Zimmer-03: Agent Zero - Consensus Engine]
    ├─ Verify API key (query Zimmer-04: Vault)
    ├─ Decode screenshot (base64 → image)
    ├─ Launch 3-model consensus in parallel:
    │   ├─ Gemini inference (call external API via Zimmer-05: Steel Stealth)
    │   ├─ Mistral inference (call external API via Zimmer-05: Steel Stealth)
    │   └─ YOLO inference (local GPU, call Zimmer-05 for TLS fingerprinting)
    ├─ Check Tier-1: Do 2/3 models agree?
    │   └─ YES: Return result (latency ~5s)
    │   └─ NO: Continue to Tier-2
    ├─ Tier-2: Wait for dual (Gemini + Mistral) with 0.9+ confidence
    │   └─ YES: Return result (latency ~8s)
    │   └─ NO: Continue to Tier-3
    ├─ Tier-3: Call CapMonster (hCaptcha specialist)
    │   └─ Return result (latency ~12s)
    ├─ Query Zimmer-15 (Surfsense) for cached solutions (semantic search)
    │   └─ If match found (score > 0.90): Use cached result
    ├─ Log result to Zimmer-10 (Postgres):
    │   └─ INSERT INTO solves (user_id, screenshot, result, latency_ms, cost_usd, timestamp)
    ├─ Log forensic data to Zimmer-07 (Stagehand) for differential feedback
    │   └─ Async POST /forensics/analyze
    └─ Return response to client
         ↓
    CLIENT RECEIVES: {result, confidence, latency_ms, cost_usd}
```

### Async Processing Pipeline (Background Jobs)

```
SOLVE COMPLETED
    ↓
[Zimmer-14: Worker Arbeiter - Job Queue]
    ├─ Job 1: Archive solve to Zimmer-10 (Postgres)
    ├─ Job 2: Update cache in Zimmer-15 (Surfsense)
    ├─ Job 3: Send differential feedback to Zimmer-07 (Stagehand) for learning
    ├─ Job 4: Update KPI dashboard (Zimmer-11: Grafana)
    ├─ Job 5: Check if detection triggered, log to Zimmer-07 for evasion learning
    └─ Job 6: Update customer usage quota in Zimmer-13 (API Brain)
```

---

## 5. RESILIENCE & FAILOVER TOPOLOGY

### Single Zimmer Failure Scenarios

**Scenario 1: Zimmer-03 (Agent Zero) crashes**
- Impact: All /api/solve requests fail
- Mitigation:
  1. Clawdbot (09) immediately routes to backup instance (Kubernetes pod)
  2. If no backup: Return 503 Service Unavailable
  3. PagerDuty SEV1 triggered (page oncall engineer)
  4. Recovery: Auto-restart container, verify health check passes

**Scenario 2: Zimmer-05 (Steel Stealth) down**
- Impact: All external API calls fail (Gemini, Mistral)
- Mitigation:
  1. Agent Zero detects circuit breaker OPEN
  2. Fallback: Skip external APIs, use YOLO + CapMonster only
  3. Solve rate slightly reduced (YOLO confidence 0.85)
  4. Cost increase (rely on expensive CapMonster)
  5. Recovery: Auto-restart, resume normal consensus

**Scenario 3: Zimmer-10 (Postgres) disk full**
- Impact: Cannot log solves to database
- Mitigation:
  1. Alert immediately (PagerDuty SEV1)
  2. Queue solves to Redis (in-memory buffer, short-term)
  3. Continue solving (no persistence temporarily)
  4. Recovery: Expand disk, resume database writes
  5. Catch-up: Replay Redis queue to Postgres (batch insert)

---

## 6. DEPLOYMENT TOPOLOGIES

### Development Topology (Single Machine - Local M1)

```
Docker Compose (1 machine):
  - Zimmer-03: Agent Zero (1 instance, 1 GPU share)
  - Zimmer-05: Steel Stealth (1 instance)
  - Zimmer-06: Skyvern (1 instance, shared)
  - Zimmer-09: Clawdbot (1 instance, port 8080)
  - Zimmer-10: Postgres (1 instance)
  - Zimmer-15: Qdrant (1 instance)
  - Zimmer-11: Grafana (1 instance)
  - Redis (implicit, shared by all)

Total Resources: 8 cores, 12GB RAM
Startup time: ~30s
Test capacity: 100 solves/minute
Cost: Free (local)
```

### Staging Topology (3 Machines - AWS/GCP)

```
Machine 1 (API & Consensus):
  - Zimmer-09: Clawdbot (load balancer)
  - Zimmer-03: Agent Zero (2 instances, 2 GPUs)
  - Zimmer-05: Steel Stealth (2 instances)

Machine 2 (Storage & Cache):
  - Zimmer-10: Postgres (with replication)
  - Zimmer-15: Qdrant (with sharding)
  - Redis (sentinel-backed HA)

Machine 3 (Analysis & Intelligence):
  - Zimmer-06: Skyvern (2 instances)
  - Zimmer-07: Stagehand (1 instance)
  - Zimmer-14: Worker (2 instances)
  - Zimmer-11: Grafana (1 instance)

Total Resources: 32 cores, 64GB RAM
Capacity: 10,000 solves/minute
Cost: ~$2,000/month
```

### Production Topology (Regional - Multi-Cloud)

```
REGION 1: EU (Frankfurt)
  [Same as staging topology, 3 machines]

REGION 2: US (Virginia)
  [Same as staging topology, 3 machines]

REGION 3: APAC (Singapore)
  [Same as staging topology, 3 machines]

CENTRAL: Monitoring & Intelligence
  - Zimmer-11: Grafana (central dashboard)
  - Zimmer-12: Evolution Optimizer (global tuning)
  - Zimmer-04: Vault (centralized secret management)
  - Zimmer-16: Supabase (real-time analytics)

Load Balancing:
  - Client requests routed by geography (GeoDNS)
  - EU clients → Frankfurt
  - US clients → Virginia
  - APAC clients → Singapore

Cross-Region Failover:
  - If Frankfurt down: Auto-route EU clients to Virginia
  - If Virginia down: Auto-route US clients to Singapore
  - If Singapore down: Auto-route APAC clients to Frankfurt

Total Resources: 96 cores, 192GB RAM
Capacity: 100,000 solves/minute
Cost: ~$20,000/month
Uptime SLA: 99.99% (multi-region redundancy)
```

---

## 7. SECURITY ARCHITECTURE & ISOLATION

### Network Security

**Internal Network (172.20.0.x/24)**:
- Docker bridge network, completely isolated from public internet
- No external inbound connections to individual zimmers
- Only Clawdbot (09) has public-facing port (8080 → 0.0.0.0:8080)
- All other zimmers: bind to localhost + internal IP only

**API Key Management**:
- All keys stored in Zimmer-04 (Vault), encrypted at rest
- Keys rotated monthly (automated by Zimmer-04)
- Audit log: Every key access logged to Zimmer-10 (Postgres)
- Revocation: Instant (remove from Vault, all zimmers check every request)

**Data Encryption**:
- In-transit: TLS 1.3 on all inter-zimmer communication
- At-rest: Encryption for Postgres (AES-256), Qdrant vectors (AES-256)
- Backup: Encrypted before upload to S3 (client-side encryption)

---

## 8. PERFORMANCE CHARACTERISTICS & SLAs

### Latency SLAs (End-to-End)

| Metric | Target | p99 | p999 |
|--------|--------|-----|------|
| Tier-1 consensus (2/3 agreement) | 5s | 8s | 12s |
| Tier-2 dual wait (8s timeout) | 8s | 10s | 15s |
| Tier-3 CapMonster | 12s | 15s | 20s |
| p50 overall latency | 5s | 5.2s | 5.5s |
| p95 overall latency | 12s | 12.5s | 13s |
| p99 overall latency | 18s | 18.5s | 19s |

### Throughput & Scalability

| Metric | Single Machine | Staging (3 machines) | Production (Regional) |
|--------|---|---|---|
| Solves/minute | 100 | 10,000 | 100,000 |
| Solves/day | 144,000 | 14.4M | 144M |
| Concurrent requests | 50 | 5,000 | 50,000 |
| API calls/day | 144K × 2.5 models | 36M calls | 360M calls |

---

## 9. HEALTH CHECK PROTOCOL & MONITORING

### Comprehensive Health Check

**Level 1: Service Health (Fast)**
```bash
curl http://172.20.0.50:8000/health
# Response: {status: "healthy", uptime_ms: 1234567}
# Timeout: 5s
# Frequency: Every 10s
```

**Level 2: Dependency Health (Moderate)**
```bash
curl http://172.20.0.50:8000/health/detailed
# Response: {
#   service: "healthy",
#   dependencies: {
#     gemini_api: "healthy",
#     mistral_api: "healthy",
#     postgres: "healthy",
#     redis: "healthy",
#     vault: "healthy"
#   }
# }
# Timeout: 10s
# Frequency: Every 60s
```

**Level 3: Full Diagnostics (Slow)**
```bash
curl http://172.20.0.50:8000/health/diagnostics
# Response: {
#   service: "healthy",
#   dependencies: {...},
#   performance: {
#     avg_latency_ms: 5200,
#     p99_latency_ms: 18900,
#     error_rate: 0.01
#   },
#   resources: {
#     memory_used_mb: 512,
#     cpu_percent: 45
#   }
# }
# Timeout: 30s
# Frequency: Every 5 min
```

---

## 10. AUTHORITY & VERSION CONTROL

**Blueprint Document**: Modul 02 - 17-Room Distributed Fortress Architecture
**Version**: 2.0 | Elite Status: 500+ Lines
**Date Created**: 2026-01-26
**Author**: Sisyphus (CEO/Engineering Lead)
**Status**: ✅ APPROVED FOR PRODUCTION
**Next Review**: 2026-02-26 (monthly review)

---

**END OF BLUEPRINT MODUL 02**
