# Google Sheets Structure

Spreadsheet ID: `1PkKrnezY7HEUKAj1njIl-USsdly4tiKtzgYy0tv8psk`

---

## Sheet 1: State

ใช้เก็บ conversation history ต่อ userId

| Column | Type | Description |
|--------|------|-------------|
| userId | string | LINE User ID (key สำหรับ filter) |
| step | string | legacy — ไม่ใช้แล้วหลัง Gemini integration |
| name | string | legacy |
| surname | string | legacy |
| phone | string | legacy |
| procedure | string | legacy |
| doctor | string | legacy |
| preferred_time | string | legacy |
| updated_at | string | legacy |
| history | string | **JSON array ของ conversation** (Gemini format) |

**หมายเหตุ:** หลัง Gemini integration (2026-06-30) คอลัมน์ที่ใช้จริงคือ `userId` และ `history` เท่านั้น คอลัมน์อื่นเป็น legacy จาก State Machine เดิม

**History format:**
```json
[
  {"role": "user", "parts": [{"text": "สวัสดี"}]},
  {"role": "model", "parts": [{"text": "สวัสดีค่ะ ยินดีต้อนรับ..."}]},
  ...
]
```
เก็บสูงสุด 20 messages ล่าสุด (10 รอบการสนทนา)

---

## Sheet 2: Appointments

เก็บนัดหมายที่สมบูรณ์ (booking_complete = true)

| Column | Type | Description |
|--------|------|-------------|
| timestamp | string | ISO 8601 เวลาที่บันทึก |
| userId | string | LINE User ID |
| name | string | ชื่อ |
| surname | string | นามสกุล |
| phone | string | เบอร์โทร |
| procedure | string | หัตถการที่ต้องการ |
| doctor | string | แพทย์ที่ต้องการ |
| preferred_time | string | วันเวลาที่สะดวก |
| status | string | "confirmed" |
| confirmed_time | string | ISO 8601 เวลายืนยัน |
