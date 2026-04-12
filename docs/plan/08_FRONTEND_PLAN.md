# Saraphan — Frontend Plan

← กลับไป [[00_MASTER_PLAN]]  
→ Security rules ดูที่ [[01_SECURITY_DESIGN]]  
→ API endpoints จะอยู่ใน backend ของแต่ละ Phase

**Tech Stack:** React 18+ / TypeScript / Ant Design 5 / EventSource (SSE)  
**ผู้รับผิดชอบ:** Frontend Developer (1 คน)  
**Effort รวม:** ~8,000-10,000 lines (ทุก phase รวมกัน)

---

## สรุปหน้าทั้งหมด (ตาม Phase)

| Phase | หน้า | ผู้ใช้ | หมายเหตุ |
|-------|------|-------|---------|
| 1 | Login | ทุกคน | mock auth ก่อน, เชื่อม NT auth ใน Phase 3 |
| 1 | Chat | ทุกคน | หน้าหลัก ถาม-ตอบ |
| 1 | ConversationHistory | ทุกคน | ดูประวัติสนทนาย้อนหลัง |
| 1 | DocumentBrowser | Manager+ | ดู/ค้นหาเอกสารในระบบ |
| 1 | DocumentUpload | Manager+ | อัปโหลดเอกสาร + เลือก classification |
| 1 | DocumentPreview | ตามสิทธิ์ | ดูเอกสารต้นฉบับ (PDF viewer) |
| 3 | PrivacyNotice | ทุกคน | PDPA notice แสดงครั้งแรกที่ login |
| 3 | AccessRequest | Director+ | ขอ/อนุมัติ cross-department access |
| 3 | ApprovalQueue | Director+/Executive | อนุมัติเอกสาร Level 3-4 |
| 3 | PermissionGrant | Director+/SysAdmin | จัดการสิทธิ์เฉพาะราย |
| 3 | NotificationCenter | ทุกคน | แจ้งเตือน pending approval, สิทธิ์หมดอายุ |
| 4 | AdminDashboard | SysAdmin | ภาพรวมระบบ |
| 4 | UserManagement | SysAdmin | จัดการ user/role/clearance |
| 4 | DepartmentManagement | SysAdmin | จัดโครงสร้างหน่วยงาน |
| 4 | DocumentManagement | SysAdmin/Director | จัดการเอกสาร batch, re-process |
| 4 | SystemConfig | SysAdmin | ตั้งค่า LLM, routing, chunking |
| 4 | Analytics | Director+/SysAdmin | query trends, knowledge gaps, cost |
| 4 | AuditLog | Director+/SysAdmin | ดู audit log (role-scoped) |
| 5 | FeedbackDashboard | SysAdmin | ดู low-rated answers, improve quality |
| 5 | DocumentVersionHistory | ตามสิทธิ์ | ดู version chain ของระเบียบ |
| 5 | GraphExplorer | Director+/SysAdmin | visualize knowledge graph (optional) |

---

## Layout หลัก

```
┌──────────────────────────────────────────────────────────┐
│ Header: Logo + Search + Notification Bell + User Menu    │
├────────────┬─────────────────────────────────────────────┤
│            │                                             │
│  Sidebar   │              Main Content                   │
│            │                                             │
│  - Chat    │                                             │
│  - เอกสาร  │                                             │
│  - อนุมัติ  │   (เปลี่ยนตามหน้าที่เลือก)                   │
│  - แจ้งเตือน│                                             │
│  - Admin ▾ │                                             │
│            │                                             │
├────────────┴─────────────────────────────────────────────┤
│ Footer: © NT + Saraphan version                          │
└──────────────────────────────────────────────────────────┘
```

**Sidebar แสดงตาม role:**
- **Staff:** Chat, เอกสาร, แจ้งเตือน
- **Manager:** + อัปโหลด
- **Director:** + อนุมัติ, สิทธิ์, Analytics (ฝ่ายตัวเอง)
- **Executive:** + Analytics (ทุกฝ่าย), Audit Log
- **SysAdmin:** + Admin (User, Department, System, Audit)

---

## Phase 1 Pages

### 1.1 Login

```
┌─────────────────────────────────┐
│         Saraphan Logo           │
│        ระบบฐานความรู้ NT          │
│                                 │
│  ┌───────────────────────────┐  │
│  │ รหัสพนักงาน               │  │
│  └───────────────────────────┘  │
│  ┌───────────────────────────┐  │
│  │ รหัสผ่าน                  │  │
│  └───────────────────────────┘  │
│                                 │
│       [ เข้าสู่ระบบ ]           │
│                                 │
│  หรือ [Login ด้วย NT SSO]       │  ← Phase 3 เพิ่ม SSO button
│                                 │
│  ล็อกอินไม่ได้? ติดต่อ IT       │
└─────────────────────────────────┘
```

- Phase 1: JWT-based, mock auth สำหรับ dev
- Phase 3: เชื่อม NT auth (LDAP/SSO)
- Failed login: แสดง error, 5 ครั้ง → lockout 15 นาที → ดู [[01_SECURITY_DESIGN#10. Session Security]]

### 1.2 Chat (หน้าหลัก)

```
┌─────────────────────────────────────────────────────────────┐
│ Sidebar              │  Chat Area                           │
│                      │                                       │
│ ┌──────────────────┐ │  ┌─────────────────────────────────┐ │
│ │ + สนทนาใหม่      │ │  │ คำถาม: ระเบียบการลาพักร้อน     │ │
│ └──────────────────┘ │  │ มีเงื่อนไขอะไรบ้าง              │ │
│                      │  └─────────────────────────────────┘ │
│ ประวัติสนทนา:        │                                       │
│ ├─ ระเบียบจัดซื้อ... │  ┌─────────────────────────────────┐ │
│ ├─ คู่มือ VPN...    │  │ [AI กำลังค้นหา...]              │ │
│ └─ นโยบาย IT...     │  │                                 │ │
│                      │  │ ตามระเบียบว่าด้วยการลาพักร้อน   │ │
│                      │  │ พ.ศ. 2567 มีเงื่อนไข...        │ │
│                      │  │                                 │ │
│                      │  │ 📎 อ้างอิง:                     │ │
│                      │  │ ├─ ระเบียบ_ลาพักร้อน.pdf p.3   │ │
│                      │  │ └─ คำสั่ง_234_2567.pdf p.1      │ │
│                      │  │                                 │ │
│                      │  │ [👍] [👎] [🔒 ลับ]              │ │
│                      │  └─────────────────────────────────┘ │
│                      │                                       │
│                      │  ┌─────────────────────────────────┐ │
│                      │  │ พิมพ์คำถาม...            [ส่ง]  │ │
│                      │  └─────────────────────────────────┘ │
└──────────────────────┴───────────────────────────────────────┘
```

**Features:**
- **Streaming response** ผ่าน SSE — แสดงคำตอบทีละคำ
- **Citation panel** — คลิกอ้างอิง → เปิด DocumentPreview ที่หน้า/section นั้น
- **Classification badge** — แสดง 🔒 ถ้าคำตอบมาจากเอกสาร Level 3+ → ดู [[01_SECURITY_DESIGN#6. Data Handling Rules]]
  - Level 1-2: แสดง source text เต็ม, copy ได้
  - Level 3: แสดง snippet สั้น, copy ไม่ได้ (disable right-click + Ctrl+C บน citation)
  - Level 4: แสดงเฉพาะชื่อเอกสาร + หน้า (reference only), ต้องไปเปิดต้นฉบับเอง
- **Feedback** — thumbs up/down + optional comment
- **Multi-turn** — ระบบจำบริบทสนทนา (Phase 5)
- **LLM routing indicator** — แสดงเล็กๆ ว่าใช้ Cloud/On-Prem LLM (สำหรับ transparency)

**Components:**
- `ChatMessage.tsx` — bubble + classification badge + copy restriction
- `CitationPanel.tsx` — clickable references, response format ตาม level
- `StreamingText.tsx` — render SSE stream
- `FeedbackButtons.tsx` — thumbs up/down + comment modal

### 1.3 ConversationHistory

- List สนทนาเก่า sort by วันที่
- ค้นหาจากข้อความใน chat เก่า
- ลบสนทนา (soft delete)
- แสดงใน sidebar ของ Chat page

### 1.4 DocumentBrowser

```
┌─────────────────────────────────────────────────────────────┐
│ [ค้นหาเอกสาร...              ] [ฝ่าย ▾] [ประเภท ▾] [ชั้นลับ ▾] │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  📄 ระเบียบว่าด้วยการจัดซื้อจัดจ้าง พ.ศ. 2568              │
│     ฝ่ายจัดซื้อ │ 🟢 เปิดเผย │ PDF │ 2568-01-15           │
│                                                             │
│  📄 คู่มือปฏิบัติงาน FTTx                                    │
│     ฝ่ายโครงข่าย │ 🟡 ใช้ภายใน │ Word │ 2567-08-01         │
│                                                             │
│  📄 นโยบายความปลอดภัยข้อมูล                                 │
│     ฝ่ายไอที │ 🔴 ลับ │ PDF │ 2568-03-10                   │
│     [ต้องขอสิทธิ์เพิ่มเพื่อดูเอกสารนี้]                       │
│                                                             │
│  [หน้า 1 / 25]                                              │
└─────────────────────────────────────────────────────────────┘
```

- แสดงเฉพาะเอกสารที่ user มีสิทธิ์เห็น metadata (ชื่อ + ฝ่าย + level)
- เอกสารที่ไม่มีสิทธิ์อ่าน: เห็นชื่อได้ แต่เปิดไม่ได้ + แสดงปุ่ม "ขอสิทธิ์" → ไป AccessRequest
- Filter: ฝ่าย, ประเภทเอกสาร, ชั้นความลับ, วันที่, สถานะ (active/archived)
- คลิกเอกสาร → DocumentPreview

### 1.5 DocumentUpload (Manager+)

```
┌─────────────────────────────────────────────────────────────┐
│ อัปโหลดเอกสาร                                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                                                       │  │
│  │          ลากไฟล์มาวางที่นี่ หรือ [เลือกไฟล์]           │  │
│  │          รองรับ: PDF, Word, Excel, รูปภาพ              │  │
│  │                                                       │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
│  ชั้นความลับ: (บังคับเลือก — ไม่มี default)                  │
│  ○ 🟢 เปิดเผย    ○ 🟡 ใช้ภายใน                             │
│  ○ 🔴 ลับ        ○ ⚫ ลับมาก                              │
│                                                             │
│  ฝ่ายเจ้าของเอกสาร: [ฝ่ายการเงิน ▾]                        │
│  ฝ่ายที่อนุญาตเพิ่มเติม: [+ เพิ่มฝ่าย]                      │
│  หมายเหตุ: [_______________________________]               │
│                                                             │
│  ⚠️ เอกสารชั้น "ลับ" ต้องรอ Director อนุมัติก่อนเข้าระบบ     │
│  ⚠️ เอกสารชั้น "ลับมาก" ต้องรอ Executive อนุมัติ             │
│                                                             │
│                               [ยกเลิก] [อัปโหลด]           │
└─────────────────────────────────────────────────────────────┘
```

- **บังคับเลือก classification** — ไม่มี default เพื่อป้องกันความผิดพลาด → ดู [[01_SECURITY_DESIGN#2. ชั้นความลับเอกสาร]]
- Staff อัปโหลดไม่ได้ (ปุ่มไม่แสดง)
- Manager อัปโหลดได้ Level 1-2 เท่านั้น
- Director+ อัปโหลดได้ทุก Level
- Upload progress bar + status tracking (processing → ready)
- Batch upload: ลากหลายไฟล์พร้อมกัน ตั้ง classification เดียวกันทั้ง batch

### 1.6 DocumentPreview

- **PDF viewer** ฝังในหน้า (pdf.js หรือ react-pdf)
- **Word rendering** แปลง docx → HTML แสดงในหน้า (mammoth.js)
- **Scanned document** แสดง OCR text คู่กับรูปสแกน
- **Classification controls:**
  - Level 1-2: ดูเต็ม + copy + download ได้
  - Level 3: ดูเต็มได้ แต่ copy/download ไม่ได้ (watermark + disable selection)
  - Level 4: ดูได้เฉพาะ user ที่มีสิทธิ์ + watermark ชื่อผู้ดู + ห้าม screenshot (best effort)
- **Metadata panel:** ชื่อ, ฝ่าย, classification, ผู้อัปโหลด, วันที่, version, อ้างอิงจาก/ถูกอ้างอิงโดย

---

## Phase 3 Pages

### 3.1 PrivacyNotice

- แสดงครั้งแรกที่ user login (ถ้ายังไม่เคย accept)
- เนื้อหา PDPA privacy notice → ดู [[01_SECURITY_DESIGN#12. PDPA Compliance Checklist]]
- ต้อง accept ก่อนใช้ระบบ
- บันทึก acceptance date + version ลง DB

### 3.2 ApprovalQueue (Director+/Executive)

```
┌─────────────────────────────────────────────────────────────┐
│ รายการรออนุมัติ                               [ทั้งหมด ▾]  │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  🔴 นโยบายเงินเดือน 2569.pdf                               │
│     ขอจัด: ลับ │ ฝ่าย HR │ โดย: สมชาย │ 2 ชม. ที่แล้ว     │
│     [ดูเอกสาร] [อนุมัติ] [ปฏิเสธ]                          │
│                                                             │
│  🔴 แผนกลยุทธ์ 2570.pdf                                    │
│     ขอจัด: ลับมาก │ สำนักกจ. │ โดย: วิชัย │ 1 วันที่แล้ว   │
│     [ดูเอกสาร] [อนุมัติ] [ปฏิเสธ]                          │
│                                                             │
│  ────────────────────────────                               │
│  อนุมัติแล้ว (ล่าสุด 7 วัน): 12 รายการ                      │
│  ปฏิเสธ: 2 รายการ                                           │
└─────────────────────────────────────────────────────────────┘
```

- Director: เห็นเฉพาะเอกสาร Level 3 ของฝ่ายตัวเอง
- Executive: เห็น Level 4 ทุกฝ่าย + Level 3 ที่ Director ส่งต่อ
- Preview เอกสารก่อนอนุมัติ
- ปฏิเสธ: ต้องระบุเหตุผล
- Workflow → ดู [[01_SECURITY_DESIGN#5. Document Lifecycle]]

### 3.3 AccessRequest

**สำหรับผู้ขอ:**
- เลือกเอกสาร/department ที่ต้องการเข้าถึง
- ระบุเหตุผล (บังคับ)
- ระบุระยะเวลา (30/60/90 วัน หรือถาวร)
- ดู status: pending → approved by my director → approved by target director → granted / rejected

**สำหรับ Director (อนุมัติ):**
- เห็น incoming requests ที่ต้องอนุมัติ
- Dual-approval: ต้องได้ทั้ง Director ฝ่ายผู้ขอ + Director ฝ่ายเจ้าของ → ดู [[01_SECURITY_DESIGN#4. Department Scope]]

### 3.4 PermissionGrant (Director+/SysAdmin)

- ค้นหา user → ดูสิทธิ์ปัจจุบัน (clearance + grants)
- Grant สิทธิ์เฉพาะราย: เลือกเอกสาร/department + ตั้ง expiry
- Revoke สิทธิ์
- ดูประวัติ grant/revoke (จาก audit log)

### 3.5 NotificationCenter

```
┌─────────────────────────────────────┐
│ 🔔 แจ้งเตือน (3 ใหม่)              │
├─────────────────────────────────────┤
│ 🆕 เอกสาร "นโยบายเงินเดือน" รอ     │
│    อนุมัติจากคุณ — 2 ชม. ที่แล้ว     │
│                                     │
│ 🆕 คำขอเข้าถึงฝ่ายการเงิน จาก      │
│    สมชาย ฝ่ายบัญชี — 1 วันที่แล้ว    │
│                                     │
│ ⚠️ สิทธิ์เข้าถึงเอกสารฝ่าย HR       │
│    จะหมดอายุใน 7 วัน                 │
│                                     │
│ ✅ เอกสาร "คู่มือ FTTx" อนุมัติแล้ว │
│    — 3 วันที่แล้ว                    │
└─────────────────────────────────────┘
```

- Badge count บน bell icon ใน header
- ประเภท notification:
  - Pending approval (สำหรับ Director/Executive)
  - Access request (สำหรับ Director)
  - สิทธิ์หมดอายุ (สำหรับ user ที่ได้ temporary grant)
  - เอกสารอนุมัติ/ปฏิเสธ (สำหรับผู้อัปโหลด)
  - System alerts (สำหรับ SysAdmin)
- Mark as read / mark all as read
- คลิก notification → ไปหน้าที่เกี่ยวข้อง

---

## Phase 4 Pages — Admin

### 4.1 AdminDashboard (SysAdmin)

```
┌─────────────────────────────────────────────────────────────┐
│ Dashboard                                                    │
├──────────────────────┬──────────────────────────────────────┤
│ วันนี้               │ สถานะระบบ                            │
│ ├─ Queries: 1,234    │ ├─ RAGFlow: 🟢 Online                │
│ ├─ Users active: 89  │ ├─ Qdrant: 🟢 Online (3 nodes)       │
│ ├─ Avg latency: 3.2s │ ├─ PostgreSQL: 🟢 Online             │
│ └─ Error rate: 0.3%  │ ├─ On-Prem LLM: 🟢 Ollama (Llama3)  │
│                      │ └─ Neo4j: 🟢 Online (Phase 5)        │
├──────────────────────┤                                       │
│ เอกสาร               │ ┌──────────────────────────────────┐ │
│ ├─ Total: 2,450      │ │ [Query Volume Chart - 7 days]    │ │
│ ├─ Processing: 3     │ │                                  │ │
│ ├─ Error: 1          │ │ ▓▓▓▓░░ ▓▓▓▓▓░ ▓▓▓▓▓▓ ▓▓▓░░░    │ │
│ └─ Pending: 5        │ │ Mon    Tue     Wed     Thu       │ │
│                      │ └──────────────────────────────────┘ │
│ Cost เดือนนี้         │                                       │
│ ├─ Cloud LLM: $890   │ ┌──────────────────────────────────┐ │
│ ├─ On-Prem: $0       │ │ [Top Unanswered Queries]         │ │
│ └─ Embedding: $45    │ │ 1. xxx (23 ครั้ง)                │ │
│                      │ │ 2. yyy (15 ครั้ง)                │ │
│                      │ └──────────────────────────────────┘ │
└──────────────────────┴──────────────────────────────────────┘
```

- Link ไป Langfuse dashboard สำหรับ detailed traces
- Quick links: User Management, Document Management, System Config

### 4.2 UserManagement (SysAdmin)

- List users + search/filter by department, role, active status
- Sync from NT directory (AD/LDAP) — button + scheduled auto-sync
- Per-user actions:
  - เปลี่ยน role/clearance level
  - Activate/deactivate
  - ดู usage stats (queries/day, last active)
  - ดู current grants
  - Force logout (revoke all sessions)
- Bulk actions: import users from CSV, batch role change

### 4.3 DepartmentManagement (SysAdmin)

- Tree view ของโครงสร้างหน่วยงาน NT
- Add/edit/deactivate department
- Set default classification level per department
- ย้าย department ใน tree (change parent)
- Sync จาก NT directory

### 4.4 DocumentManagement (SysAdmin/Director)

- List เอกสารทั้งหมด (SysAdmin) หรือฝ่ายตัวเอง (Director)
- Filter: status (processing/ready/error/archived), classification, department
- Batch actions:
  - Re-process failed documents
  - Change classification (ต้องอนุมัติ → ApprovalQueue)
  - Archive/unarchive
  - Delete (soft delete, ต้องอนุมัติถ้า Level 3-4)
- Processing queue: ดูเอกสารที่กำลัง process + status

### 4.5 SystemConfig (SysAdmin)

- **LLM Settings:** Cloud API keys, On-Prem LLM endpoints, default model
- **Routing Rules:** keywords ที่ trigger on-prem routing, classification thresholds
- **Search Config:** chunk size, overlap, RRF k value, blend weights, query expansion toggle
- **Rate Limits:** per-user, per-IP limits
- **Maintenance:** clear cache, rebuild embeddings, re-index
- ทุกการเปลี่ยนแปลง → audit log

### 4.6 Analytics (Director+/SysAdmin)

- **Director** เห็นเฉพาะ stats ของฝ่ายตัวเอง
- **Executive/SysAdmin** เห็นทั้งหมด

**Dashboards:**
- Top queries (popular topics)
- Unanswered queries (knowledge gaps) — เอกสารที่ยังขาด
- Document usage — เอกสารไหนถูกอ้างอิงบ่อย
- User activity — queries per department, active users
- Cost tracking — per LLM provider, per department
- Satisfaction — average feedback score over time
- Response quality — RAGAS metrics trend (ถ้า run scheduled)

### 4.7 AuditLog (Director+/SysAdmin)

- **Director** ดู log เฉพาะฝ่ายตัวเอง
- **Executive** ดู log ทั้งหมด
- **SysAdmin** ดู log ทั้งหมด + system events
- Filter: user, action type, date range, resource type
- Export: CSV/JSON (SysAdmin only)
- การเข้าดู audit log → บันทึกใน audit log อีกชั้น (view_audit action) → ดู [[01_SECURITY_DESIGN#9. Audit Log Requirements]]

---

## Phase 5 Pages

### 5.1 FeedbackDashboard (SysAdmin)

- List คำตอบที่ได้ thumbs down พร้อม query + response + citation
- Filter by: rating, date, department, topic
- Admin สามารถ: mark as reviewed, add note, add to golden set
- Knowledge gap detection: queries ที่ระบบตอบว่า "ไม่พบข้อมูล" → alert ให้เพิ่มเอกสาร

### 5.2 DocumentVersionHistory

- แสดง version chain ของระเบียบ/คำสั่ง
- Timeline view: version 1 → version 2 (แก้ไข) → version 3 (ยกเลิก)
- เชื่อมกับ GraphRAG: SUPERSEDES / AMENDS edges → ดู [[07_PHASE_5_GRAPHRAG]]
- Diff view (ถ้า text-based): highlight ส่วนที่เปลี่ยนแปลง

### 5.3 GraphExplorer (Optional, Director+/SysAdmin)

- Interactive graph visualization (D3.js หรือ vis-network)
- แสดง nodes (ระเบียบ/คำสั่ง) + edges (อ้างอิง, ยกเลิก, แก้ไข)
- คลิก node → ดูรายละเอียด + ไป DocumentPreview
- Filter by: department, topic, relation type
- Priority ต่ำ — ทำถ้ามีเวลาเหลือ

---

## Shared Components

| Component | ใช้ในหน้า | หมายเหตุ |
|-----------|----------|---------|
| `AppLayout.tsx` | ทุกหน้า | Header + Sidebar + Main + Footer |
| `RoleGuard.tsx` | ทุกหน้า | ซ่อน/แสดง element ตาม user role |
| `ClassificationBadge.tsx` | Chat, Documents, Preview | 🟢🟡🔴⚫ badge ตาม level |
| `DepartmentSelect.tsx` | Upload, Search, Admin | dropdown + tree select |
| `UserSearch.tsx` | PermissionGrant, UserManagement | autocomplete user search |
| `DateRangePicker.tsx` | Analytics, AuditLog | Ant Design DatePicker |
| `PaginatedTable.tsx` | ทุก list page | Ant Design Table + pagination + sort |
| `ConfirmModal.tsx` | Delete, Approve, Reject | confirmation + reason input |
| `LoadingState.tsx` | ทุกหน้า | skeleton loading, spinner |
| `ErrorBoundary.tsx` | ทุกหน้า | error fallback UI |
| `NotificationBell.tsx` | Header | badge count + dropdown |
| `CopyRestriction.tsx` | Chat, Preview | disable copy สำหรับ Level 3-4 |

---

## Frontend Effort Estimate (ตาม Phase)

| Phase | หน้า/Components | ประมาณ Lines | สัปดาห์ |
|-------|----------------|-------------|---------|
| Phase 1 | Login, Chat, ConversationHistory, DocumentBrowser, Upload, Preview + shared components | ~3,500-4,000 | สัปดาห์ 5-6 |
| Phase 3 | PrivacyNotice, ApprovalQueue, AccessRequest, PermissionGrant, NotificationCenter | ~1,500-2,000 | สัปดาห์ 12-13 |
| Phase 4 | AdminDashboard, UserMgmt, DeptMgmt, DocMgmt, SystemConfig, Analytics, AuditLog | ~3,000-4,000 | สัปดาห์ 16-18 |
| Phase 5 | FeedbackDashboard, VersionHistory, GraphExplorer (optional) | ~800-1,200 | สัปดาห์ 20-21 |
| **Total** | **21 pages + 12 shared components** | **~8,800-11,200** | |

---

## API Contract Summary

Backend (FastAPI) endpoints ที่ frontend ต้องใช้ — implement ตามแต่ละ Phase:

**Phase 1 — [[03_PHASE_0_1_CORE]]:**
- `POST /api/v1/auth/login` → JWT token
- `POST /api/v1/chat` → SSE stream (question → answer)
- `GET /api/v1/conversations` → list
- `GET /api/v1/conversations/{id}/messages` → messages
- `POST /api/v1/documents/upload` → upload + classify
- `GET /api/v1/documents` → list (filtered by ACL)
- `GET /api/v1/documents/{id}` → metadata
- `GET /api/v1/documents/{id}/content` → file content (PDF/rendered HTML)
- `GET /api/v1/health` → system status

**Phase 3 — [[05_PHASE_3_SECURITY]]:**
- `POST /api/v1/auth/sso` → SSO login
- `GET/POST /api/v1/approvals` → approval queue + actions
- `GET/POST /api/v1/access-requests` → access request CRUD
- `GET/POST /api/v1/permissions` → grant/revoke
- `GET /api/v1/notifications` → notification list
- `POST /api/v1/privacy-notice/accept` → accept PDPA notice

**Phase 4 — [[06_PHASE_4_PRODUCTION]]:**
- `GET/POST/PUT /api/v1/admin/users` → user CRUD
- `GET/POST/PUT /api/v1/admin/departments` → department CRUD
- `GET/PUT /api/v1/admin/config` → system config
- `GET /api/v1/admin/analytics/*` → various analytics endpoints
- `GET /api/v1/admin/audit-log` → audit log (role-scoped)

**Phase 5 — [[07_PHASE_5_GRAPHRAG]]:**
- `GET /api/v1/feedback` → feedback list
- `GET /api/v1/documents/{id}/versions` → version chain
- `GET /api/v1/graph/explore` → graph data for visualization
- `POST /api/v1/feedback/{id}/review` → admin review feedback
