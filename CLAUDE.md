# Alonsi Clinic LINE Bot — Project Guide

## ภาพรวม

chatbot LINE คลินิกทันตกรรมซี-สไมล์ ยะลา ทำหน้าที่:
1. ตอบคำถามทั่วไป (ราคา, เวลา, ทีมแพทย์, สิทธิ/ประกัน) ด้วย Gemini AI
2. รับจองนัด → บันทึกลง Google Sheets → แจ้ง admin ทาง LINE
3. Handoff ให้หมอ/admin รับสาย → บอทหยุดตอบชั่วคราว → รีเซ็ตเมื่อพิมพ์ "จบแล้วค่ะ"

## Stack

| ชั้น | เครื่องมือ | รายละเอียด |
|------|-----------|------------|
| Automation | n8n Cloud | csmileyala.app.n8n.cloud |
| AI | Gemini 2.5 Flash | via REST API + $vars.GEMINI_API_KEY |
| Messaging | LINE Messaging API | Webhook → n8n |
| Storage | Google Sheets | State (history/handoff) + Appointments |

## Credentials — ทั้งหมดอยู่ใน n8n (ไม่มีใน code)

| ชื่อ | เก็บที่ | ใช้โดย |
|------|---------|--------|
| LINE token | n8n Header Auth credential "LINE Messaging API" | 5 HTTP nodes |
| LINE Channel Secret | n8n Variable `LINE_CHANNEL_SECRET` | C2 Verify Signature node |
| Gemini API Key | n8n Variable `GEMINI_API_KEY` | Gemini API URL expression |

**ห้ามเพิ่ม credential ใดๆ ลงใน workflow JSON เด็ดขาด**

## โครงสร้าง Repo

```
workflow/
  n8n-workflow.json   ← snapshot ล่าสุด (ไม่มี credential)
sheets/
  schema.md           ← โครงสร้าง Google Sheets (State + Appointments)
memory/
  MEMORY.md           ← index
  *.md                ← project context สำหรับ Claude
CLAUDE.md             ← ไฟล์นี้
```

## Key Nodes (IDs ใน workflow JSON)

| Node | ID | หน้าที่ |
|------|----|---------|
| Verify LINE Signature | `a1b2c3d4-e5f6-7890-abcd-ef1234567890` | C2: HMAC verify ด้วย $vars.LINE_CHANNEL_SECRET |
| If (skip filter) | `70ed36d9-c685-44da-8c09-a724d7f8a970` | M1: กรอง userId ว่างออก |
| Gemini API | `23b05e53-052b-4de9-9fde-9f55744abf3c` | เรียก Gemini, continueOnFail=true |
| Parse Gemini Response | `936121a9-e14c-425f-9648-ada291500fb7` | C3: fallback เมื่อ Gemini ล้ม |
| Reply to LINE | `8e92ed83-d5ef-48b5-a9b9-07e5f14f3ac8` | ส่งข้อความกลับ user |
| Push Notify Admin | `aa000005` | แจ้ง admin เมื่อ handoff ถูกทริกเกอร์ |
| Reply Handoff End | `aa000009` | "บอทคุกกี้กลับมาแล้วค่ะ" เมื่อรีเซ็ต |

## State Machine

```
ข้อความเข้า
  → Verify Signature (C2)
  → Parse LINE Message
  → If skip (userId ว่าง?) → ออก
  → Read State (Google Sheets)
  → Check Handoff Mode?
      ├── handoff active + ไม่ใช่ "จบแล้วค่ะ" → หยุด (M2: silent)
      ├── handoff active + "จบแล้วค่ะ" → Reset → "บอทคุกกี้กลับมา"
      └── normal mode
            → trigger word? → Push Notify Admin + Reply Handoff Start
            └── ไม่ใช่ → Gemini → Parse → If1
                    ├── booking_complete=true → Append Appointment → Delete State
                    └── booking_complete=false → Save State → Reply
```

## Trigger Words (Handoff)

ใน Check Handoff node — คำที่ทำให้บอทส่งต่อให้หมอ/admin:
`ปวดมาก`, `เร่งด่วน`, `ฉุกเฉิน`, `คุยกับหมอ` ฯลฯ (ดู node ใน n8n)

## Fixes ที่ทำไปแล้ว

| Fix | รายละเอียด | สถานะ |
|-----|-----------|--------|
| C1 | LINE token → n8n Header Auth credential | ✅ |
| C2 | HMAC verify enforced, ใช้ $vars.LINE_CHANNEL_SECRET | ✅ |
| C3 | Gemini fail → fallback reply + เบอร์คลินิก | ✅ |
| M1 | If node leftValue `={{ $json.userId }}` (เพิ่ม =) | ✅ |
| M2 | If Is Reset false branch → disconnect (silent handoff) | ✅ |

## Pending

| Fix | รายละเอียด |
|-----|-----------|
| C4 | Delete State ควร match ด้วย userId ไม่ใช่ row_number (race condition) |
| M3 | Webhook idempotency — deduplicate by event.message.id |
| M4 | Validate appointment fields ก่อน Append |

## วิธีอัปเดต Workflow

### ผ่าน n8n API (แนะนำ)

```bash
# GET
curl -H "X-N8N-API-KEY: <key>" https://csmileyala.app.n8n.cloud/api/v1/workflows/GFdRhaxbgbxvIPjw

# PUT (fields ที่ต้องมี)
# { name, nodes, connections, settings: {executionOrder: "v1"}, staticData }
```

n8n API key ต้องสร้างใหม่ที่ csmileyala.app.n8n.cloud → Settings → API Keys (มีวันหมดอายุ)

### หลัง deploy ทุกครั้ง → sync local JSON

```bash
# Download live → save
curl -s -H "X-N8N-API-KEY: <key>" \
  https://csmileyala.app.n8n.cloud/api/v1/workflows/GFdRhaxbgbxvIPjw \
  | python3 -c "import sys,json; d=json.load(sys.stdin); [d.pop(k,None) for k in ('id','meta','createdAt','updatedAt','versionId','sharedWithProjects','tags')]; print(json.dumps(d,indent=2,ensure_ascii=False))" \
  > workflow/n8n-workflow.json
git add workflow/n8n-workflow.json && git commit -m "sync: update workflow snapshot"
```

## System Prompt ของ Gemini

อยู่ใน node "Build Gemini Request" → `parameters.jsCode` → ตัวแปร `systemPrompt`

เนื้อหาหลัก: บอทชื่อ "คุกกี้", ข้อมูลคลินิก, ทีมแพทย์, ราคาบริการ, สิทธิ/ประกัน, ขั้นตอนจองนัด, format JSON

## หมายเหตุสำคัญ

- n8n ใช้ `$vars.X` ใน Code nodes (ไม่ใช่ `$env.X`) — Pro plan เท่านั้น
- LINE webhook ส่ง `rawBody` มาด้วย — ใช้อันนี้ใน HMAC verify ห้ามใช้ re-serialized JSON
- Gemini ต้องมี `continueOnFail: true` และ Parse node ต้อง handle error output ด้วย
- Google Sheets มี 2 sheet: `State` (history/handoff per userId) และ `Appointments`
