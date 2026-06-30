# Alonsi Clinic LINE Bot

LINE chatbot สำหรับคลินิกทันตกรรมซี-สไมล์ — รับนัดหมายและตอบคำถามด้วย AI (Gemini)

---

## ภาพรวมระบบ

```
LINE User
  ↓ ส่งข้อความ
LINE Official Account (Messaging API)
  ↓ webhook POST
n8n Cloud (csmileyala.app.n8n.cloud)
  Workflow: "Alonsi Clinic LINE Bot"
  ├── Webhook1                        → รับ event จาก LINE (rawBody enabled)
  ├── Verify LINE Signature           → ตรวจ X-Line-Signature (HMAC-SHA256)
  ├── Parse LINE Message              → แยก userId / replyToken / message
  ├── If                              → กรอง non-message events ออก
  ├── Read State                      → Google Sheets: ดึง history ของ userId
  ├── Build Gemini Request            → สร้าง request payload สำหรับ Gemini
  ├── Gemini API                      → เรียก gemini-2.5-flash
  ├── Parse Gemini Response           → แยก replyText / booking_complete / appointment
  ├── If1                             → เช็ค booking_complete
  │   ├── false → Save State (userId + history + updated_at) → Reply to LINE
  │   └── true  → Append row (Appointments) → Delete State row → Reply to LINE
```

---

## Configuration

### Credentials ที่ต้องมี

| Service | ใช้ทำอะไร | เก็บที่ไหน |
|---------|-----------|-----------|
| LINE Channel Access Token | ส่ง reply กลับหา user | n8n → node "Reply to LINE" → Header Authorization |
| Gemini API Key | เรียก Gemini AI | n8n → node "Gemini API" → URL parameter `key=` |
| Google Sheets OAuth | อ่าน/เขียน State + Appointments | n8n Credentials → "Google Sheets account" |

### Environment

| Item | Value |
|------|-------|
| n8n instance | csmileyala.app.n8n.cloud |
| Workflow ID | GFdRhaxbgbxvIPjw |
| Webhook URL | `https://csmileyala.app.n8n.cloud/webhook/Line-webhook` |
| Google Spreadsheet | `1PkKrnezY7HEUKAj1njIl-USsdly4tiKtzgYy0tv8psk` |
| Gemini model | `gemini-2.5-flash` |

---

## ข้อมูลคลินิก (System Prompt)

- **ชื่อ:** คลินิกทันตกรรมซี-สไมล์
- **ที่อยู่:** 9/1 ถ.อาคารสงเคราะห์ ต.สะเตง อ.เมือง จ.ยะลา 95000
- **โทร:** 0863655805
- **Google Maps:** https://maps.app.goo.gl/jR6J5MPPcrRm5stt5
- **เวลาทำการ:** จ-ศ 10:00-20:00 | ส-อ/หยุด 10:00-16:00 (โทรเช็คก่อนเสมอ)
- **ทีมแพทย์:** หมอซี, หมอแพร, หมอต๋อย, หมอโฟร์, หมอช่อ (เวลาลงคลินิกไม่ตรงกัน — โทรเช็ค)
- **บอทชื่อ:** คุกกี้

### บริการและราคา

| บริการ | ราคา |
|--------|------|
| อุดฟัน | 800-1,600 บาท |
| ถอนฟัน | 600-1,200 บาท |
| ขูดหินปูน | 800-1,600 บาท |
| ผ่าฟันคุด | 3,000 บาท |
| ฟันปลอมแบบถอดได้ (พลาสติก) | เริ่มต้น 3,000 บาท |
| รากฟันเทียม | เริ่มต้น 35,000 บาท |
| รักษารากฟัน (หน้า/กรามน้อย/กราม) | 5,000 / 7,000 / 12,000 บาท |
| จัดฟัน (ปรึกษา) | 100 บาท |
| จัดฟัน (พิมพ์ฟัน+Xray) | 2,000 บาท |
| จัดฟัน (ติดเครื่องมือ จ่ายสด) | 10,000 + เดือนละ 1,000 บาท |
| จัดฟัน (ผ่อน) | 5,000 / 5,000 / เดือนละ 1,000 บาท |
| รีเทนเนอร์ | 2,500/ชิ้น (2 ชิ้น 5,000 บาท) |

---

## Gemini Response Format

บอทตั้ง `responseMimeType: 'application/json'` — Gemini ตอบ JSON เสมอ:

```json
// ระหว่างสนทนา
{"replyText": "...", "booking_complete": false, "appointment": null}

// เมื่อนัดเสร็จ
{
  "replyText": "สรุปนัดหมาย...",
  "booking_complete": true,
  "appointment": {
    "name": "...", "surname": "...", "phone": "...",
    "procedure": "...", "doctor": "...", "preferred_time": "..."
  }
}
```

---

## Flow หลัก

### If1 Logic
- `booking_complete !== true` → **TRUE [0]** → Save History → Reply
- `booking_complete === true` → **FALSE [1]** → Append Appointments → Delete State row → Reply

### Save State
เขียนลง State sheet: `userId` + `history` (JSON string, max 20 messages) + `updated_at` (ISO timestamp)

**Auto-cleanup:** cron ทุกคืนเที่ยงคืน (เวลาไทย) ลบ row ที่ `updated_at` เก่ากว่า 24 ชั่วโมง

### Conversation History (Gemini format)
```js
[
  { role: 'user', parts: [{ text: 'ข้อความ user' }] },
  { role: 'model', parts: [{ text: 'ข้อความ bot' }] }
]
```

---

## ไฟล์ในโปรเจกต์นี้

```
alonsi-clinic-bot/
├── README.md                    ← ไฟล์นี้
├── workflow/
│   └── n8n-workflow.json        ← workflow JSON (sanitized, ไม่มี API key จริง)
├── sheets/
│   └── structure.md             ← schema ของ Google Sheets
└── .claude/
    └── agents/
        └── line-bot.md          ← sub-agent + บทเรียนที่เรียนรู้มา
```

---

## บทเรียนที่เรียนรู้ (สรุป)

ดูรายละเอียดเต็มใน `.claude/agents/line-bot.md` — มี 8 บทเรียน

| # | ปัญหา | สาเหตุ | วิธีแก้ |
|---|-------|--------|---------|
| 1 | LINE แสดง `\n` เป็นตัวอักษร | ใช้ `'...\n...'` แทน template literal | ใช้ backtick เสมอ |
| 2 | JSON body invalid | ใช้ "Fields Below" แทน "JSON" | ใช้ "Using JSON" + expression เดียว |
| 3 | Referenced node doesn't exist | ชื่อ node ไม่ตรงใน expression | ตรวจชื่อ node ก่อนอ้างถึง |
| 4 | Invalid reply token | token หมดอายุ | test ด้วย LINE message จริงเท่านั้น |
| 5 | Workflow ไม่รับ message | workflow ยัง Unpublish | กด Publish + เช็ค webhook ON |
| 6 | บัญชีชื่อเหมือน 2 อัน | Channel name แก้ไม่ได้ | แก้ที่ LINE OA Manager |
| 7 | State ค้าง | row userId ยังอยู่ใน Sheet | ลบ row นั้นด้วยมือ |
| 8 | **If1 true→Append bug** | เส้น true ต่อเกินไป Append | ลบเส้น true→Append ออก |

---

## การเปลี่ยนแปลง System Prompt

แก้ที่ n8n → node **"Build Gemini Request"** → แก้ JS code ในส่วน `systemPrompt = ...`

หรือใช้ n8n API:
```bash
# GET workflow
curl -H "X-N8N-API-KEY: <key>" https://csmileyala.app.n8n.cloud/api/v1/workflows/GFdRhaxbgbxvIPjw

# PUT workflow (อัปเดต)
curl -X PUT -H "X-N8N-API-KEY: <key>" -H "Content-Type: application/json" \
  -d @workflow.json \
  https://csmileyala.app.n8n.cloud/api/v1/workflows/GFdRhaxbgbxvIPjw
```

---

## ประวัติการพัฒนา

| วันที่ | เหตุการณ์ |
|--------|-----------|
| 2026-06-24 | เริ่มต้น chatbot ด้วย State Machine (switch/case) |
| 2026-06-25 | พบและแก้ bug If1 true→Append connection |
| 2026-06-30 | เพิ่ม Gemini AI (gemini-2.5-flash) แทน State Machine |
| 2026-06-30 | เปลี่ยนชื่อบอทเป็น "คุกกี้" |
| 2026-06-30 | เพิ่ม LINE Signature Verification (HMAC-SHA256) |
| 2026-06-30 | Auto-cleanup: Save State เขียน updated_at + cron ล้าง session ค้าง 24 ชม. |
| 2026-06-30 | เพิ่ม Google Maps link ใน system prompt (แสดงแผนที่เมื่อถามที่อยู่) |
| 2026-06-30 | เพิ่ม FAQ sheet + logging คำถามที่ไม่รู้จัก |
