**ภาษา:** ภาษาไทย | [English](02-write-adr.en.md)

# งานที่ 2: เขียน ADR สำหรับความต้องการใหม่ (ประมาณ 20 นาที)

> **เชื่อมโยงกับการบรรยาย:** T3 (GPT-4 แบบ zero-shot ให้ผลที่สอดคล้องกัน แต่ *ครบถ้วนน้อยกว่ามนุษย์* ส่วน few-shot ช่วยลดช่องว่างได้มาก), AICH3 (บริบท), สัปดาห์ที่ 10 (message broker)
> **คำถาม:** ADR ที่ AI เขียนดีขึ้นแค่ไหน เมื่อให้ตัวอย่างและบริบทจริงแก่ AI

อ่าน [สถานการณ์ (ภาษาไทย)](../docs/scenario.th.md) ก่อน: ความต้องการ **R-12** ว่าด้วยสิ่งที่เกิดขึ้นหลังจากลูกค้าสั่งซื้อสินค้า

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่าง โดยแทนที่ `<paste R-12 here>` ด้วยข้อความ R-12 ภาษาอังกฤษที่อยู่ในกรอบอ้างอิงของ scenario

**Run a: ไม่มีตัวอย่าง (zero-shot)** → [`outputs/2a-adr.md`](outputs/2a-adr.md)
```
Follow prompts/2-write-adr.md
run: a
context: none
requirement: <paste R-12 here>
language: Thai
```

**Run b: มีตัวอย่าง + บริบท** → [`outputs/2b-adr.md`](outputs/2b-adr.md)
```
Follow prompts/2-write-adr.md
run: b
context: docs/scenario.md, docs/adr/0001-use-redis-for-cart-storage.md, system/online-boutique/src/checkoutservice/main.go
requirement: <paste R-12 here>
language: Thai
```

> บรรทัด `language: Thai` ทำให้ agent เขียน ADR เป็นภาษาไทย (หัวข้อของ template และชื่อทางเทคนิคยังคงเป็นภาษาอังกฤษ) หากต้องการภาษาอังกฤษให้ลบบรรทัดนี้ **ใช้ภาษาเดียวกันทั้ง Run a และ Run b** เพื่อให้เปรียบเทียบกันได้

> ใน Run a ให้ตรวจว่า agent ไม่ได้อ่านไฟล์อื่นนอกจาก template จริง ๆ หาก agent เปิด `docs/scenario.md` ให้บันทึกไว้ใน prompt log เพราะจะทำให้การเปรียบเทียบเปลี่ยนไป

## เปรียบเทียบ a กับ b

| เกณฑ์ | Run a | Run b |
|---|---|---|
| ใช้ตัวเลขจาก scenario หรือไม่ (latency, ช่วงพีก, ข้อจำกัดของ MailFast) | ไม่ใช้เลย ระบุชัดว่าไม่มีข้อมูล volume, latency target, SLA และ rate limit ของ MailFast แล้วย้ายไปไว้ใน Open questions | ใช้ครบ: p95 ≤ 500 ms (ตอนนี้ประมาณ 300 ms), 200 orders/s, MailFast p95 1.5 s / p99 8 s / 0.5 % / 50 emails/s, loyalty ภายใน 1 นาที, ทีม 4 คนรู้ RabbitMQ 2 คน, Kubernetes + self-hosted และนำไปคำนวณต่อ (backlog ประมาณ 9,000 อีเมล ใช้เวลาระบายประมาณ 3 นาที) |
| ทางเลือกที่พิจารณา (ระบุ) | 1. Sync calls 2. Async fire-and-forget 3. Broker pub/sub 4. Transactional outbox + broker เลือกข้อ 3 แต่**ไม่ระบุผลิตภัณฑ์** (เลื่อนการเลือกออกไปเพราะไม่มีข้อมูลทีมและ hosting) | 1. Sync gRPC chain 2. In-process goroutines 3. RabbitMQ pub/sub 4. Kafka เลือก **RabbitMQ** ด้วยเหตุผลจาก scenario (ทีมเคยใช้, open-source, self-hosted) ตัด managed broker ออกเพราะข้อจำกัดด้าน operations ส่วน outbox ไม่ได้เป็นทางเลือก แต่ไปอยู่ใน Risks/Open questions เพราะ `checkoutservice` ไม่มี store |
| จัดการแต้มสะสมแบบ "ครั้งเดียวเท่านั้น" และ rate limit ของ MailFast หรือไม่ อย่างไร | **ทั่วไป และเป็นแค่เงื่อนไข:** exactly-once ถูกตั้งเป็นคำถามว่า "จำเป็นหรือไม่" และเสนอ idempotent consumer ด้วย order ID ส่วน rate limit เขียนว่า "ถ้ามี" ให้ consumer คุมอัตราการส่งเอง ไม่มีตัวเลข | **เป็นรูปธรรม:** `loyaltyservice` เก็บ `order_id` พร้อม unique constraint ใน transaction เดียวกับการให้แต้ม และ ack หลัง commit (exactly-once effect บน at-least-once delivery) แยก queue ต่อ action ให้ email backlog ไม่ทำให้แต้มช้า, consumer อีเมลจำกัด ≤ 50/s ทุก replica, retry + backoff + DLQ และมี metric/alert ที่ 1 นาที |
| มีสิ่งที่แต่งขึ้นเองหรือไม่ (ตัวเลข เครื่องมือ หรือข้อเท็จจริงที่ไม่มีใน scenario) | ไม่มีตัวเลขที่แต่งขึ้น ผลิตภัณฑ์ถูกพูดถึงแค่เป็นตัวอย่าง **ข้อสังเกต:** หัวข้อ exactly-once และ rate limit ของ MailFast รั่วมาจากใบงานที่ agent อ่านก่อนรัน (ดู prompt log แถว 3) ไม่ได้มาจาก requirement | ตัวเลขส่วนใหญ่ตรงกับ scenario มีการแต่งเพิ่มเล็กน้อย: เครื่องมือ **Locust** และ **quorum queues** (ยกเป็น "e.g." ไม่มีในบริบท), ค่า publish "a few ms" (ติด [Inferred] ไว้แล้ว) และเลขบรรทัดใน `main.go` (เช่น l. 230–280, l. 273) ซึ่งต้องตรวจเทียบไฟล์จริง |

## รับผิดชอบ ADR ฉบับสุดท้ายของตัวเอง

นิสิตคือสถาปนิก ให้คัดลอก `outputs/2b-adr.md` ไปที่ `docs/adr/0002-<short-title>.md` แล้วแก้ไข**ด้วยตัวเอง**จนพร้อมจะลงชื่อรับรอง ห้ามให้ agent ทำส่วนนี้

- แก้ไขสิ่งที่ผิดหรือแต่งขึ้นเอง และเพิ่มสิ่งที่ขาดหายไป
- ตัดสินใจขั้นสุดท้ายด้วยตัวเอง นิสิตสามารถไม่เห็นด้วยกับ AI ได้

นิสิตจะต้องปกป้อง ADR นี้ต่อหน้าคณะกรรมการรีวิวในส่วน B จึงต้องอธิบายทุกการตัดสินใจใน ADR ได้

| # | สิ่งที่เปลี่ยนจากร่างของ AI | เหตุผล |
|---|---|---|
| 1 | เพิ่มทางเลือกที่ 5 **Transactional outbox + broker** ในตาราง pros/cons และส่วน "Why this option" (นำมาจาก 2a) | ร่าง b พูดถึงความเสี่ยงที่ event อาจหาย แต่ไม่ได้ชั่งน้ำหนัก outbox เป็นทางเลือกจริง พอเพิ่มเข้ามาก็อธิบายได้ว่าทำไม*ยังไม่*เลือก (`checkoutservice` ไม่มี store และทีมมีแค่ 4 คน) และจะเปลี่ยนไปใช้เมื่อไร |
| 2 | เขียน Decision ใหม่ให้มีเหตุผล "ทำไมไม่เลือก" ครบทุกทางเลือก (1, 2, 4, 5) ไม่ใช่มีแค่ "Why not Kafka" | ต้องปกป้อง ADR ต่อหน้าคณะกรรมการ จึงต้องตอบได้ว่าทำไมไม่เลือกแต่ละทาง และแสดงว่าทางที่ 3 เป็นทางเดียวที่ผ่าน driver 1–4 พร้อมกัน |
| 3 | เพิ่มข้อยกเว้นของ **fraud scoring**: ถ้าต้อง block order จะต้องอยู่ก่อน payment และต้องมี ADR แยก | ร่าง b ถือว่า fraud scoring เป็นแค่ subscriber ตัวใหม่ แต่การตรวจ fraud ที่ต้องหยุด order ไม่สามารถทำแบบ async หลังจ่ายเงินแล้วได้ |
| 4 | ทุก consequence ระบุ "quality attribute:" ชัดเจน และเพิ่มข้อ scalability/deployability และ observability | AGENTS.md กำหนดให้ทุก consequence ผูกกับ quality attribute ส่วนการ trace ข้าม async hop เป็นต้นทุนจริงที่ร่าง b ไม่ได้พูดถึง |
| 5 | เพิ่ม risk 3 ข้อ (อีเมลซ้ำตอน retry, schema ของ event เปลี่ยน, trace ขาด) ใส่ schema version ใน `OrderPlaced` และเพิ่ม validation 2 ข้อ (extensibility check, lost-event metric) | ทำให้ driver ด้าน modifiability และ "ไม่มี event หาย" ทดสอบได้จริง ไม่ใช่แค่อ้างลอย ๆ |
| 6 | ตัดหัวข้อ **Open questions** ออก ให้หัวข้อตรงกับ `docs/adr/template.md` | template ไม่มีหัวข้อนี้ **แต่ยังมีคำว่า "(see Open questions)" ค้างอยู่ 3 จุด (บรรทัด 7, 72, 112) ต้องแก้หรือลบเองใน ADR** |
