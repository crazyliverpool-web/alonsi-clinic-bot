---
name: user-prefs
description: วิธีทำงานร่วมกับผู้ใช้ในโปรเจกต์นี้
metadata:
  type: feedback
---

## การทำงาน

**ตอบภาษาไทย เสมอ** — กระชับ ไม่ formal ไม่ต้องสรุปท้ายทุก response

**Plan before execute**
งานที่มีผลจริง (แก้ n8n, push git, เปลี่ยน workflow) → แสดงแผนก่อน รอ confirm แล้วค่อยทำ
งานอ่านอย่างเดียว (ดู JSON, list files, fetch logs) → ทำได้เลย

**Why:** ผู้ใช้ต้องการ control ทุกขั้นตอน ไม่ให้ Claude ลงมือก่อนโดยไม่แสดงแผน

**Opus design, Sonnet execute**
- ถ้ามีการตัดสินใจ design ใหม่หรือ architecture → แนะนำให้ switch Opus ก่อน
- งาน mechanical (รันคำสั่ง, แก้ node, อัปเดต JSON) → Sonnet ทำได้เลย
- ผู้ใช้ซีเรียสเรื่อง usage/cost

**How to apply:** งานซับซ้อน หรือต้องเลือก approach → บอกให้ switch Opus ก่อน ไม่ลงมือเองทันที

## n8n workflow — วิธีแก้

ใช้ n8n REST API + curl เท่านั้น (Python urllib มี SSL error บน macOS)
PUT body ต้องมีแค่: `{name, nodes, connections, settings: {executionOrder}, staticData}`
ห้าม include: `id, versionId, meta, createdAt, updatedAt, sharedWithProjects, tags`

**Why:** n8n PUT 400 "additional properties" ถ้าส่ง field ที่ไม่รองรับ
