**ภาษา:** ภาษาไทย | [English](01-recover-architecture.en.md)

# งานที่ 1: กู้คืนสถาปัตยกรรม (ประมาณ 15 นาที)

> **เชื่อมโยงกับการบรรยาย:** T7 การกู้คืนสถาปัตยกรรม, AICH2 (การตามรอย), AICH3 (บริบท) และข้อค้นพบจาก JSS ว่า 93% ของงานวิจัยไม่เคยตรวจสอบผลลัพธ์จาก AI
> **คำถาม:** AI กู้คืนสถาปัตยกรรมได้มากแค่ไหน และขึ้นอยู่กับหลักฐานที่ AI ได้อ่านมากเพียงใด

**ระหว่างทำงานนี้ ห้ามเปิด repository ต้นฉบับของ Online Boutique** เพราะ README ของต้นฉบับมีแผนภาพสถาปัตยกรรมที่นิสิตกำลังกู้คืนอยู่

ในแต่ละ run ให้เริ่ม **แชตหรือ session ใหม่** แล้วส่งข้อความด้านล่างให้ agent โดย agent จะทำตาม [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) และเขียนผลลัพธ์ไว้ที่ `exercise/outputs/1<run>-architecture.md`

**Run A: เฉพาะ interface**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interface + ไฟล์ตั้งค่าการ deploy**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

ระหว่างที่ agent ทำงาน ให้สังเกตว่า agent เปิดไฟล์ใดบ้าง หาก agent อ่านไฟล์นอกเหนือจากหลักฐานที่อนุญาต ให้บันทึกไว้ใน prompt log

## ผลลัพธ์

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

เปิดแต่ละไฟล์ด้วย **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) เพื่อดูแผนภาพ

## ตรวจสอบ Run B ด้วยตัวเอง (ส่วนที่สำคัญที่สุด)

ตรวจสอบ dependency ในตารางของ Run B **อย่างน้อยห้ารายการ** เทียบกับไฟล์จริง โดยเปิดไฟล์ที่ AI อ้างถึงและหาหลักฐาน นิสิตสามารถตรวจซอร์สโค้ดของ service ได้ด้วย เช่น `src/checkoutservice/main.go`

| ผู้เรียก → ผู้ถูกเรียก | หลักฐานที่ AI อ้าง | ยืนยันแล้วหรือไม่ ✅ / ❌ / ❓ | สิ่งที่นิสิตพบ |
|---|---|---|---|
| `checkoutservice` → `paymentservice` | `checkoutservice.yaml` · `PAYMENT_SERVICE_ADDR` | ✅ | มี env var ใน manifest (บรรทัด 61) และโค้ดเรียกจริง: `src/checkoutservice/main.go` บรรทัด 116 (`mustMapEnv`) และบรรทัด 370 `NewPaymentServiceClient(...).Charge(...)` |
| `checkoutservice` → `emailservice` | `checkoutservice.yaml` · `EMAIL_SERVICE_ADDR` | ✅ | `main.go` บรรทัด 380 `SendOrderConfirmation(...)` ซึ่งถูกเรียกจาก `PlaceOrder` บรรทัด 273 ถ้าเกิด error จะแค่ log warning |
| `checkoutservice` → `cartservice` | `checkoutservice.yaml` · `CART_SERVICE_ADDR` | ✅ | `main.go` บรรทัด 325 `GetCart` และบรรทัด 333 `EmptyCart` (label "Reads/empties cart" ถูกต้อง) |
| `recommendationservice` → `productcatalogservice` | `recommendationservice.yaml` · `PRODUCT_CATALOG_SERVICE_ADDR` | ✅ | `src/recommendationservice/recommendation_server.py` บรรทัด 131 อ่าน env var และบรรทัด 73 เรียก `ListProducts` |
| `cartservice` → `redis-cart` | `cartservice.yaml` · `REDIS_ADDR=redis-cart:6379` | ✅ | `src/cartservice/src/Startup.cs` บรรทัด 29 อ่าน `REDIS_ADDR` ส่วน `redis-cart` ใช้ `image: redis:alpine` (`cartservice.yaml` บรรทัด 118) |
| `frontend` → `shippingservice` | `frontend.yaml` · `SHIPPING_SERVICE_ADDR` | ✅ / ❓ | เชื่อมต่อจริง แต่ `src/frontend/rpc.go` บรรทัด 88 เรียกแค่ `GetQuote` ส่วน label "Gets shipping quote" ถูก แต่ Run B เองระบุว่าไม่มีหลักฐานว่าเรียก RPC ใด |

- **สิ่งที่ผิดหรือขาดหายไป** ใน Run B (dependency ที่ไม่มีจริง, dependency ที่ขาดไป, container ที่น่าสงสัย):
 - **container ที่น่าสงสัย:** `loadgenerator` ถูกวาดไว้ในขอบเขตระบบ ทั้งที่เป็นเครื่องมือทดสอบและถูก comment ออกจาก `kustomization.yaml` (บรรทัด 24) จึงไม่ใช่ส่วนหนึ่งของระบบ production
 - **dependency ที่ขาดไป:** `frontend` → `shoppingassistantservice` (`SHOPPING_ASSISTANT_SERVICE_ADDR` ใน `frontend.yaml` และ `src/frontend/main.go`) และมีโฟลเดอร์ `src/shoppingassistantservice/` อยู่จริง แต่ Run B ตัดออกจากแผนภาพ (ระบุไว้ใน Gaps)
 - **ภาษาที่ใช้เขียนแต่ละ service** ไม่ได้ระบุ เพราะหลักฐานไม่มี แต่ซอร์สโค้ดแสดงว่าภาษาต่างกัน เช่น checkout เป็น Go, recommendation เป็น Python, cart เป็น C# และ ad เป็น Java/Gradle
- label ของ `Rel` บางเส้นระบุ RPC เกินกว่าที่มีหลักฐาน เช่น "Quotes and ships" แต่ตรวจแล้วส่วนใหญ่ถูกต้อง
- ไม่พบ dependency ที่ไม่มีอยู่จริง ทั้ง 17 เส้นมี env var รองรับ

## สะท้อนคิด (ข้อละ 2–3 ประโยค)

1. เปรียบเทียบ A กับ B: agent **ไม่สามารถ**รู้อะไรได้จากไฟล์ `.proto` เพียงไฟล์เดียว และเพราะเหตุใด
   ใครเรียกใคร, ว่ามี `frontend` เป็นทางเข้า, data store (`redis-cart`) และการ deploy (port, LoadBalancer) เพราะ `.proto` เป็นแค่ interface contract ที่บอกว่าแต่ละ service *ให้บริการ*อะไร ไม่ได้บอกว่าใครเป็น client หรือมีอะไรอยู่เบื้องหลัง ผลคือ Run A ไม่มีเส้น dependency เลย ส่วน Run B ได้ 17 เส้นจาก env var `*_ADDR` ใน manifest

2. ข้อผิดพลาด**ข้อใดข้อหนึ่ง**ที่จะทำให้สมาชิกใหม่ของทีมเข้าใจผิดมากที่สุด หากเผยแพร่แผนภาพนี้โดยไม่ตรวจสอบ
 แผนภาพของ Run A แสดง gRPC service 9 ตัวที่ไม่มีเส้นเชื่อมกันเลย ไม่มี `frontend` และไม่มี data store คนอ่านจะคิดว่า service แยกขาดจากกัน ทั้งที่ `checkoutservice` เรียก 6 service แบบ synchronous ทุกครั้งที่สั่งซื้อ ความเสี่ยงหลักด้าน availability จึงหายไปจากแผนภาพ ข้อมูลที่ "ไม่มีหลักฐาน" กลายเป็น "ไม่มีอยู่จริง" เมื่อวาดออกมาเป็นภาพว่าง ๆ
