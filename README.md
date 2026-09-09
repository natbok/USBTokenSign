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
- **Header:**
  ```http
  Authorization: Bearer {{token}}
  ```
  _(บังคับใช้ JWT Bearer Token ทุกกรณี เพื่อความถูกต้องของ Audit Log และการระบุตัวตนระบบต้นทาง)_

### พารามิเตอร์ (Form-Data)

| พารามิเตอร์        | ประเภท | จำเป็น? | คำอธิบาย                                                         |
| :----------------- | :----: | :-----: | :--------------------------------------------------------------- |
| **`Pdf`**          |  File  | **ใช่** | ไฟล์ PDF ต้นฉบับที่ต้องการเซ็น                                   |
| **`Xml`**          |  File  |   ไม่   | ไฟล์ XML e-Tax (ระบบจะฝังเป็น Attachment ใน PDF ให้)             |
| **`SourceSystem`** |  Text  |   ไม่   | ระบุชื่อระบบผู้ส่ง เพื่อบันทึกใน Log (เช่น `"HIS"`, `"Finance"`) |

---

## 3. สิ่งที่ได้รับกลับมา (Response)

### กรณีสำเร็จ (`200 OK`)

- ได้รับข้อมูลเป็น **ไฟล์ PDF ที่เซ็นเสร็จแล้วทันที** (`application/pdf`)
- เมื่อเปิดด้วย Adobe Acrobat Reader จะพบ:
  - แถบ **Blue Ribbon** แจ้งว่าเอกสารผ่านการ Certified
  - มีตราประทับเวลาสากล (Timestamp)
  - มีไฟล์ XML แนบอยู่ด้านใน (หากส่งมาด้วย)

### กรณีผิดพลาด (Error)

- **`400 Bad Request`**: ไม่ได้แนบไฟล์ PDF
- **`401 Unauthorized`**: ไม่ได้ใส่ Token หรือ Token ไม่ถูกต้อง
- **`503 Service Unavailable`**: เครื่อง Server ไม่ได้เสียบ USB Token หรือระบบมองไม่เห็น Token

---

## 4. ตัวอย่างการเรียกใช้งาน (cURL)

```bash
curl -X POST "{{baseUrl}}/api/Sign" \
  -H "Authorization: Bearer {{token}}" \
  -F "Pdf=@invoice.pdf" \
  -F "Xml=@invoice.xml" \
  -F "SourceSystem=HIS" \
  --output "invoice_signed.pdf"
```

---
