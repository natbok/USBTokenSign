# คู่มือการทำงาน API ลงนามดิจิทัล (`{{baseUrl}}/api/Sign`)

API นี้ใช้สำหรับส่งไฟล์ PDF มาเซ็นลายเซ็นดิจิทัลด้วย **USB Token** บนเครื่อง Server โดยอัตโนมัติ (พร้อมแนบไฟล์ XML และประทับเวลาสากล RFC 3161)

---

## 1. ลำดับการทำงาน (Workflow)

```
[ระบบต้นทาง (HIS/ERP/App)]
          │
          │  1. ส่งไฟล์ PDF (+ XML ถ้ามี)
          ▼
   [API: /api/Sign]
          │
          │  2. ตรวจสอบสิทธิ์ (JWT Bearer Token)
          │  3. ฝังไฟล์ XML ลงใน PDF (ถ้าแนบมา)
          │  4. ปลดล็อก USB Token ดึง Private Key มาเซ็น
          │  5. ดึงเวลาสากล (Timestamp) มาประทับ
          │  6. บันทึกประวัติ (Audit Log)
          │
          │  7. ส่งไฟล์ PDF ที่เซ็นแล้วกลับไปทันที
          ▼
[ได้ไฟล์ PDF ที่ลงนามสมบูรณ์]
```

---

## 2. ข้อมูลการเรียกใช้งาน (Request)

- **Method:** `POST`
- **URL:** `{{baseUrl}}/api/Sign`
- **Content-Type:** `multipart/form-data`
- **Headers:**
  ```http
  Authorization: Bearer {{token}}
  User-Urgent: true   # (ทางเลือกเสริม) กำหนดเป็นคำขอด่วนเพื่อแซงคิวเข้าทำก่อน (หรือใช้ X-Urgent: true)
  ```
  _(บังคับใช้ JWT Bearer Token ทุกกรณี เพื่อความถูกต้องของ Audit Log และการระบุตัวตนระบบต้นทาง)_

### พารามิเตอร์ (Form-Data)

| พารามิเตอร์        | ประเภท  | จำเป็น? | คำอธิบาย                                                         |
| :----------------- | :-----: | :-----: | :--------------------------------------------------------------- |
| **`Pdf`**          |  File   | **ใช่** | ไฟล์ PDF ต้นฉบับที่ต้องการเซ็น                                   |
| **`Xml`**          |  File   |   ไม่   | ไฟล์ XML e-Tax (ระบบจะฝังเป็น Attachment ใน PDF ให้)             |
| **`SourceSystem`** |  Text   | **ใช่** | ระบุชื่อระบบผู้ส่ง เช่น `"HIS"`, `"Cashier-OPD"`                  |
| **`IsUrgent`**     | Boolean |   ไม่   | กำหนดเป็น `true` สำหรับงานด่วนที่ต้องการผลลัพธ์ $\le$ 4.30 วินาที (แซงคิว Normal ทั้งหมด) |

---

## 3. สิ่งที่ได้รับกลับมา (Response)

### กรณีสำเร็จ (`200 OK`)

- ได้รับข้อมูลเป็น **ไฟล์ PDF ที่เซ็นเสร็จแล้วทันที** (`application/pdf`)
- มี Header `Server-Timing` แสดงสถิติเวลาประมวลผลแต่ละขั้นตอน
- เมื่อเปิดด้วย Adobe Acrobat Reader จะพบ:
  - แถบ **Blue Ribbon** แจ้งว่าเอกสารผ่านการ Certified
  - มีตราประทับเวลาสากล (Timestamp)
  - มีไฟล์ XML แนบอยู่ด้านใน (หากส่งมาด้วย)

### กรณีผิดพลาด (Error)

- **`400 Bad Request`**: ไม่ได้แนบไฟล์ PDF หรือไม่ได้ระบุ `SourceSystem`
- **`401 Unauthorized`**: ไม่ได้ใส่ Token หรือ Token ไม่ถูกต้อง / หมดอายุ
- **`429 Too Many Requests`**: คิวประมวลผลเต็ม (คิวด่วนเต็ม 3 คิว หรือคิวปกติเต็ม 15 คิว) ระบบจะส่งกลับทันทีใน 5 ms เพื่อป้องกันไม่ให้รอนานจน Timeout:
  ```json
  {
    "status": "error",
    "code": 429,
    "message": "Urgent queue is full (current: 3, max: 3). Please retry in 2-3 seconds or submit as normal priority.",
    "is_urgent": true,
    "current_queue": 3,
    "max_queue": 3,
    "retry_after_seconds": 3
  }
  ```
- **`503 Service Unavailable`**: เครื่อง Server ไม่ได้เสียบ USB Token หรือระบบมองไม่เห็น Token

---

## 4. ตัวอย่างการเรียกใช้งาน (cURL)

### 4.1 แบบปกติ (Normal Priority):
```bash
curl -X POST "{{baseUrl}}/api/Sign" \
  -H "Authorization: Bearer {{token}}" \
  -F "Pdf=@invoice.pdf" \
  -F "Xml=@invoice.xml" \
  -F "SourceSystem=HIS" \
  --output "invoice_signed.pdf"
```

### 4.2 แบบด่วนพิเศษ (Urgent Priority $\le$ 4.30s):
```bash
curl -X POST "{{baseUrl}}/api/Sign" \
  -H "Authorization: Bearer {{token}}" \
  -H "User-Urgent: true" \
  -F "Pdf=@invoice.pdf" \
  -F "SourceSystem=Cashier-OPD" \
  -F "IsUrgent=true" \
  --output "invoice_signed.pdf"
```

---

## 5. Endpoints ตรวจสอบสถานะคิว (Queue Monitoring)

- **ดูสถานะคิวทั้งหมด:** `GET {{baseUrl}}/api/Sign/queue/status`
- **ดูเฉพาะคิวด่วน:** `GET {{baseUrl}}/api/Sign/queue/info-urgent` เพื่อดูว่า isUrgentAvailable เป็น true หรือไม่ 

*(อ่านรายละเอียดระบบคิวและ SLA ฉบับสมบูรณ์ได้ที่: [PriorityQueueArchitecture.md](../API/PriorityQueueArchitecture.md))*

