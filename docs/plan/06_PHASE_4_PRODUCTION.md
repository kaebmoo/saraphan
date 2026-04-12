# Saraphan — Phase 4: Production Hardening

← กลับไป [[00_MASTER_PLAN]]  
← ก่อนหน้า [[05_PHASE_3_SECURITY]]  
→ ถัดไป [[07_PHASE_5_GRAPHRAG]]  
→ Admin dashboard spec ดูที่ [[08_FRONTEND_PLAN#Admin Dashboard]]

---

## Phase 4: Production Hardening (สัปดาห์ 15-18)

**เป้าหมาย:** รองรับ 1,000+ users, มี monitoring, admin จัดการระบบได้

### Task 4.1: Load Testing + Performance Optimization (สัปดาห์ 15)
- [ ] ทดสอบด้วย Locust/k6:
  - 100 concurrent users, 10 queries/sec
  - Target: p95 latency < 5 seconds (รวม retrieval + generation)
- [ ] Bottleneck analysis + optimization:
  - Embedding cache (Redis): cache vector ของ query ที่ซ้ำ
  - Search result cache (Redis): cache top-k results, TTL 5 min
  - Connection pooling (RAGFlow, Qdrant, PostgreSQL)
  - Async everywhere (FastAPI + httpx)
- [ ] Qdrant optimization: HNSW parameters tuning, quantization ถ้าจำเป็น
- **ผู้รับผิดชอบ:** DevOps + Backend 1
- **เครื่องมือ:** Locust (open-source), Redis
- **ต้องเขียนเอง:** cache layer ~200-300 lines

### Task 4.2: Observability — Langfuse (สัปดาห์ 15-16)
- [ ] Deploy Langfuse (self-hosted Docker)
- [ ] Integrate: trace ทุก LLM call
  - Input prompt, output, tokens, latency, cost
  - Retrieval step: query, results, scores
  - User feedback linkage
- [ ] Dashboard:
  - Daily query volume
  - Avg latency breakdown (retrieval vs generation)
  - Token cost per department
  - Error rate
  - User satisfaction (feedback score)
- **ผู้รับผิดชอบ:** DevOps + Backend 2
- **เครื่องมือ:** Langfuse (open-source, self-hosted)
- **ต้องเขียนเอง:** instrumentation ~200 lines

### Task 4.3: Admin Dashboard (สัปดาห์ 16-18)
- [ ] Document Management:
  - อัปโหลด batch, จัด category/department
  - ดู processing status, re-process failed documents
  - ตั้ง classification level per document
- [ ] User Management:
  - Sync จาก NT directory
  - จัดสิทธิ์ per department/role
  - ดู usage per user
- [ ] System Config:
  - LLM provider settings
  - Routing rules (cloud/onprem)
  - Chunking parameters
  - Embedding model selection
- [ ] Analytics:
  - Top queries (popular topics)
  - Unanswered queries (knowledge gaps)
  - Document usage (เอกสารไหนถูกอ้างอิงบ่อย)
  - Cost tracking per LLM provider
- **ผู้รับผิดชอบ:** Frontend + Backend 2
- **ต้องเขียนเอง:** ทั้งหมด ~4,000-5,000 lines (React + API)

### Task 4.4: High Availability Setup (สัปดาห์ 17-18)
- [ ] Qdrant: 3-node cluster (HA mode)
- [ ] PostgreSQL: primary + replica
- [ ] FastAPI: 2+ instances behind load balancer
- [ ] RAGFlow: evaluate scaling options
- [ ] Automated backup:
  - PostgreSQL: daily pg_dump
  - Qdrant: snapshot to S3/NAS
  - Document storage: replicated
- [ ] Health check + auto-restart (Docker restart policy / K8s)
- [ ] Disaster recovery plan document
- **ผู้รับผิดชอบ:** DevOps
- **ต้องเขียนเอง:** Docker Compose (production) + backup scripts

### Milestone 4: Production Ready
- [ ] p95 latency < 5s ที่ 100 concurrent users
- [ ] Langfuse dashboard แสดงข้อมูลครบ
- [ ] Admin dashboard ใช้งานได้
- [ ] HA setup ทดสอบ failover สำเร็จ
- [ ] Backup + restore ทดสอบสำเร็จ

---

