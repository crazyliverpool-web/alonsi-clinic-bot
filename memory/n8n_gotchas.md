---
name: n8n-gotchas
description: ข้อควรระวังเมื่อแก้ n8n workflow ผ่าน API
metadata:
  type: feedback
---

## $vars vs $env

ใน n8n Cloud Pro, Code nodes ต้องใช้ `$vars.KEY_NAME` ไม่ใช่ `$env.KEY_NAME`
`$env` ใช้ไม่ได้บน Cloud — จะได้ค่าว่างเสมอ

**Why:** n8n Cloud ไม่ expose environment variables ให้ workflow โดยตรง ต้องใช้ n8n Variables (Pro plan)

## PUT Workflow — Fields ที่ต้องส่ง

```json
{
  "name": "...",
  "nodes": [...],
  "connections": {...},
  "settings": {"executionOrder": "v1"},
  "staticData": null
}
```

ห้ามส่ง: `id`, `versionId`, `meta`, `createdAt`, `updatedAt`, `sharedWithProjects`, `tags`
→ n8n จะ return 400 "additional properties" ทันที

## continueOnFail

HTTP Request node ที่อาจล้ม (เช่น Gemini API) ต้องตั้ง `"continueOnFail": true` ที่ node level
เมื่อ fail → output ออกมาใน main path แต่ `json` จะมี `error` field แทน response ปกติ
→ Parse node ต้องเช็ค `if (geminiBody.error || !geminiBody.candidates)` ก่อนเสมอ

## LINE Signature Verification

rawBody ต้องใช้จาก `$input.item.json.rawBody` — ห้าม re-serialize JSON ใหม่
HMAC: `crypto.createHmac('sha256', secret).update(rawBody, 'utf8').digest('base64')`
Header: `x-line-signature` (lowercase)

## n8n Variables API

POST /api/v1/variables → สร้าง variable ใหม่
n8n อาจ normalize key case ในบางเวอร์ชัน → verify ด้วย GET /api/v1/variables หลัง create

## curl ไม่ใช่ Python urllib

Python urllib มี SSL/certificate error กับ n8n Cloud บน macOS
ใช้ curl เสมอ: `curl -s -w '\n%{http_code}' -X PUT -H 'X-N8N-API-KEY: ...' -H 'Content-Type: application/json' -d @file.json URL`
