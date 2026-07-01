---
name: project-bot-state
description: สถานะ bot คลินิก Alonsi — fixes ที่ deploy แล้วและ pending items
metadata:
  type: project
---

## สถานะล่าสุด (2026-07-01)

Bot ชื่อ "คุกกี้" ทำงานบน n8n Cloud (csmileyala.app.n8n.cloud) workflow ID: `GFdRhaxbgbxvIPjw`

**Why:** คลินิกทันตกรรมซี-สไมล์ ยะลา ต้องการ chatbot รับจองนัด + ตอบคำถามอัตโนมัติ

### Fixes ที่ deploy แล้ว ✅

| Fix | รายละเอียด |
|-----|-----------|
| C1 | LINE token ย้ายเข้า n8n Header Auth credential (ชื่อ "LINE Messaging API") |
| C2 | HMAC-SHA256 verify บังคับแล้ว ใช้ $vars.LINE_CHANNEL_SECRET |
| C3 | Gemini fail → reply fallback + โทร 0863655805 |
| M1 | If skip filter leftValue แก้เป็น `={{ $json.userId }}` (เพิ่ม =) |
| M2 | If Is Reset false branch disconnect แล้ว (silent handoff) |
| Gemini key | เก็บเป็น n8n Variable `GEMINI_API_KEY` ไม่อยู่ใน JSON |
| System prompt | เพิ่ม section สิทธิ/ประกัน (บัตรทอง/ประกันสังคม/ฟันคุด) |

### Pending ❌

| Fix | รายละเอียด | Priority |
|-----|-----------|----------|
| C4 | Delete State ควร filter userId ไม่ใช่ row_number — race condition เมื่อมีคนพิมพ์พร้อมกัน | High |
| M3 | Webhook idempotency — LINE ส่ง event ซ้ำได้ ต้อง deduplicate by event.message.id | Medium |
| M4 | Validate appointment fields ครบก่อน Append Sheets | Low |

### n8n Variables (ทั้งหมด encrypted)

- `LINE_CHANNEL_SECRET` — ใช้ใน C2 Verify Signature
- `GEMINI_API_KEY` — ใช้ใน Gemini API URL expression

**How to apply:** เมื่อมาทำงานต่อ — C4 คือ priority ถัดไป แก้ที่ Delete State node ให้ filter sheet ด้วย userId แทน row_number
