# GoGoTrip Project Overview (สำหรับผู้มาใหม่)

เอกสารนี้สรุปภาพรวมโครงสร้างระบบ **Frontend / Backend / AI** ของโปรเจกต์นี้ เพื่อให้เริ่มทำงานได้เร็วขึ้น

## 1) ภาพรวมระบบ

โปรเจกต์นี้เป็นระบบแชตบอทท่องเที่ยวที่ใช้ Django เป็นแกนหลัก และเชื่อมกับ LINE Messaging API เพื่อรับ-ส่งข้อความลูกค้า โดยสถาปัตยกรรมหลักคือ

- **Backend (Django + DRF):** จัดการข้อมูลทริป การจอง การชำระเงิน และ API CRUD
- **AI Layer (LangChain + LLM):** วิเคราะห์ intent, คุยเชิงทั่วไป, ตอบข้อมูลทริป, และตรวจสอบสลิป
- **Frontend:** ฝั่งนี้ใน repo มีเพียงหน้าเดโม `chat/templates/chat.html` และยังมี frontend จริงที่ลิงก์ภายนอก (URL อยู่ในข้อความที่ AI สรุปทริป)

## 2) โครงสร้างโค้ดที่ควรรู้ก่อน

- `tripbot/settings.py` — ตั้งค่า environment, database, LINE token/secret, CORS, static/media
- `tripbot/urls.py` — route หลัก เช่น `webhook/line/` และ `api/`
- `chat/models.py` — schema ธุรกิจทั้งหมด (Trip, Booking, Payment, LineUser, LineMessage ฯลฯ)
- `chat/serializers.py` — แปลง model <-> JSON สำหรับ DRF
- `chat/api_views.py` — ViewSet และ custom action ต่าง ๆ
- `chat/line_webhook.py` — จุดรับ event จาก LINE และ orchestration การตอบกลับ
- `chat/agent.py` — orchestration ของ AI agent
- `chat/tools.py` — tools ที่ AI เรียกใช้เพื่อ query ข้อมูลจริง
- `chat/prompts.py` — system prompts สำหรับ classify / trip / friendly / payment verify

### 2.1 อธิบายไฟล์ต่าง ๆ แบบเร็ว

| ไฟล์ | ทำหน้าที่ | ทำไมต้องรู้ |
|---|---|---|
| `tripbot/settings.py` | รวม config หลักของระบบ เช่น DB, ENV, LINE, CORS, static/media | เวลา run ไม่ขึ้นหรือ config เพี้ยน มักต้องเริ่มดูไฟล์นี้ก่อน |
| `tripbot/urls.py` | ประกาศ route หลักของโปรเจกต์ | ใช้เช็กว่าคำขอจาก frontend/LINE/API เข้ามาที่ endpoint ไหน |
| `chat/models.py` | นิยามโครงสร้างข้อมูลธุรกิจทั้งหมด | ถ้าไม่เข้าใจไฟล์นี้ จะตาม flow การจอง การชำระเงิน และ history แชตได้ยาก |
| `chat/serializers.py` | แปลง model เป็น JSON และรับ JSON เข้า model | สำคัญเวลาทำ API หรือ debug ว่าข้อมูลที่ส่งออก/รับเข้ามี shape แบบไหน |
| `chat/api_views.py` | รวม DRF ViewSet และ custom actions | เป็นจุดหลักของ backend CRUD เช่น trips, bookings, payments |
| `chat/urls.py` | map ViewSet ของ app `chat` เข้ากับ router | ช่วยดูว่า resource ไหน expose เป็น API บ้าง |
| `chat/line_webhook.py` | รับ event จาก LINE, เรียก AI, ส่งข้อความกลับ, จัดการ state | ถือเป็นหัวใจของ flow production เพราะข้อความลูกค้าเข้ามาทางนี้จริง |
| `chat/agent.py` | รวม logic ฝั่ง AI orchestration | ใช้ตัดสินว่าจะ classify, ตอบแบบคุยทั่วไป, ใช้ tools, หรือวิเคราะห์สลิป |
| `chat/tools.py` | ฟังก์ชันที่ AI ใช้ดึงข้อมูลจริงจากระบบ | เป็นตัวเชื่อมระหว่าง LLM กับข้อมูลจากฐานข้อมูล |
| `chat/prompts.py` | เก็บ system prompts/instructions ของ AI | เวลา AI ตอบไม่ตรงโจทย์ ส่วนใหญ่ต้องกลับมาดูไฟล์นี้ |
| `chat/templates/chat.html` | หน้าเว็บแชตเดโมแบบง่าย | ช่วยดูภาพรวม frontend ที่มีใน repo นี้ แม้ยังไม่ใช่ production UI หลัก |
| `chat/tests.py` | จุดเริ่มสำหรับเขียน automated tests | ตอนนี้ยังมีน้อย จึงเป็นพื้นที่ที่ควรขยายต่อในอนาคต |
| `chat/management/commands/` | management commands สำหรับงานทดสอบ/seed/mock | มีประโยชน์เวลาต้องรันงานช่วยเหลือผ่าน `manage.py` |
| `chat/migrations/` | ประวัติ schema migration ของ Django | ใช้ตรวจว่าฐานข้อมูลเปลี่ยนอะไรไปบ้างและสัมพันธ์กับ model อย่างไร |

## 3) Backend (Django/DRF)

### 3.1 Routing
- Route ระดับโปรเจกต์:
  - `POST /webhook/line/` รับ webhook จาก LINE
  - `api/` สำหรับ DRF router endpoints
- DRF ใช้ `DefaultRouter` register หลาย resource เช่น trips, bookings, payments, line-users/messages

### 3.2 Data Model ที่สำคัญ
- **User / Customer / Admin** — โครงสร้างผู้ใช้หลักและบทบาท
- **Trip** — ข้อมูลแพ็กเกจท่องเที่ยว (location/category/date/price/capacity/content)
- **Image** — ภาพต่อทริป รองรับภาพปก 1 รูปผ่าน flag `image_thumbnail`
- **Booking** — การจองทริป (group_size, total_price, status)
- **Payment** — การชำระเงิน + สถานะตรวจสอบสลิป + ผู้ยืนยัน
- **LineUser / LineMessage** — mapping ผู้ใช้ LINE และ history ข้อความเข้าออก
- **ChatbotSession** — log คำถาม-คำตอบของบอทระดับ session
- **Rating** — feedback หลังจบทริป

### 3.3 API Layer
- แต่ละ resource มี ViewSet พร้อม filter query params
- มี custom action สำคัญ เช่น
  - `TripViewSet.available`
  - task ส่ง feedback หลังจบทริป (`daily_trip_feedback_cron`, `test_trip_feedback`)
  - `PaymentViewSet.upload_slip`, `verify_payment`, `pending_verification`

## 4) AI Layer (LangChain + LLM)

### 4.1 ตัวหลักใน `chat/agent.py`
- `classify_agent()` — แยก intent ว่าเป็น friendly_chat หรือ admin_chat
- `friendly_agent()` — ตอบคุยทั่วไป
- `admin_agent()` — สร้าง OpenAI tools agent แล้วเรียก tools จริงจากฐานข้อมูล
- `payment_verify_agent()` — ใช้ภาพสลิปแบบ multimodal เพื่อคัดว่าภาพดูเป็นสลิปโอนหรือไม่
- `run_agent()` — ฟังก์ชันศูนย์กลางที่รับข้อความผู้ใช้เข้ามา 1 ครั้ง แล้วตัดสินใจเส้นทางทำงานทั้งหมด เช่น
  - เรียก `classify_agent()` เพื่อแยก intent ก่อน
  - ถ้าเป็น `friendly_chat` จะส่งต่อไป `friendly_agent()`
  - ถ้าเป็น `admin_chat` จะส่งต่อไป `admin_agent()` ให้เรียก tools/query ข้อมูลจริง
  - สุดท้ายคืนผลลัพธ์มาตรฐานในรูป `response_type`, `response_content`, `response_meta` เพื่อให้ webhook เอาไปสร้างข้อความตอบกลับ (text/flex)

### 4.2 Tools ที่ AI ใช้ใน `chat/tools.py`
- `get_trips()` — query ทริปตามเงื่อนไข (จังหวัด/ภูมิภาค/ราคา/วันที่/หมวด)
- `get_equipments()` — ดึงรายการอุปกรณ์
- `get_date()` — utility คำนวณวันที่
- `get_payments()` — ดึงรายการชำระเงินตามสถานะ

> จุดสำคัญ: AI ไม่ได้ “เดา” อย่างเดียว แต่มีการเรียก tools เพื่อดึงข้อมูลจริงจาก DB

## 5) LINE Webhook Flow (หัวใจของโปรดักชัน)

1. LINE ส่ง event -> `line_webhook()`
2. แยกประเภทข้อความ (text/image/sticker/...)
3. บันทึก message เข้า `LineMessage`
4. ถ้าเป็น text ปกติ -> `run_agent()`
5. ตาม `response_type` สร้างข้อความตอบกลับ:
   - text
   - trip_list (Flex carousel)
   - booking_verify (Flex confirm)
6. บันทึกข้อความขาออกลง `LineMessage`
7. ถ้าเป็นรูปและ user อยู่สถานะ `pending_payment` -> เรียก `payment_verify_agent()` แล้วสร้าง booking/payment

## 6) Frontend ใน repo นี้

- มีหน้าเดโมแบบง่ายที่ `chat/templates/chat.html`
- ใช้ JavaScript ส่ง `POST /ask/` (endpoint นี้ยังไม่เห็นใน URL config ปัจจุบัน)
- สื่อว่าไฟล์นี้น่าจะเป็น demo/legacy มากกว่า UI หลักใน production
- ใน `agent.py` มีลิงก์ frontend ภายนอกสำหรับรายละเอียดทริป (`.../blog/{{trip_id}}`)

## 7) สิ่งสำคัญที่ต้องรู้ (Quick Start Mental Model)

- **Domain หลัก:** Trip -> Booking -> Payment -> (Feedback)
- **Channel หลัก:** LINE webhook เป็น entrypoint จริง
- **AI orchestration:** classify ก่อน แล้วค่อยเลือก friendly/admin flow
- **State machine ฝั่ง LINE:** ใช้ `line_user.user_status` และ `user_metadata` คุมสถานะคุย (idle, pending_payment, waiting_for_reason ฯลฯ)
- **API + Admin operation:** ฝั่ง CRUD อยู่ที่ DRF ViewSet ทั้งหมด

## 8) ความเสี่ยง/ข้อควรระวังที่ผู้มาใหม่ควรรู้

- `ALLOWED_HOSTS = ['*']` เหมาะ dev เท่านั้น ต้อง tighten ก่อน production
- `CORS_ALLOW_ALL_ORIGINS = True` ควรจำกัด origin ตอนขึ้นจริง
- ใน `chat.html` เรียก `/ask/` แต่ route นี้ไม่ปรากฏใน `tripbot/urls.py`
- dependency ค่อนข้างใหญ่ ควรแยก requirement ตาม runtime/ai/dev ในอนาคต

## 9) แนะนำเส้นทางเรียนรู้ต่อ (ลำดับที่คุ้มสุด)

1. **อ่าน flow end-to-end:** `chat/line_webhook.py` -> `chat/agent.py` -> `chat/tools.py`
2. **เข้าใจ domain model:** `chat/models.py` (Trip/Booking/Payment/LineMessage)
3. **ลองยิง API:** ใช้ Postman ทดสอบ `/api/trips`, `/api/bookings`, `/api/payments`
4. **ปรับ prompt อย่างระวัง:** เปลี่ยนทีละจุดใน `chat/prompts.py` แล้วดูผลกับ conversation จริง
5. **เพิ่ม tests:** เริ่มจาก webhook handler และ tools query
6. **แยก config env ชัดเจน:** dev/staging/prod และตรวจ secret handling

## 10) เช็กลิสต์สำหรับคนเริ่ม contribute

- [ ] เซ็ต `.env` ครบ (DB, LINE, OpenAI/OpenRouter)
- [ ] `python manage.py migrate` ผ่าน
- [ ] เข้าใจ response schema ที่ AI ส่ง (`response_type`, `response_content`, `response_meta`)
- [ ] เข้าใจ Flex message helper ที่ใช้ตอบใน LINE
- [ ] ทดสอบกรณี text + image (pending_payment) + feedback flow
