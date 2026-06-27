# DT-508-Body-Tracking

โปรแกรมตรวจจับและวาด Pose Skeleton บนวิดีโอหรือกล้องเว็บแคมแบบ Real-time โดยใช้ MediaPipe Pose Landmarker และ OpenCV

---

## ภาพรวมโปรแกรม

โปรแกรมอ่านแต่ละ frame จากแหล่งวิดีโอ แล้วส่งให้ MediaPipe ตรวจจับตำแหน่ง landmark 33 จุดบนร่างกาย จากนั้นวาดเส้นเชื่อมต่อและจุดกลมลงบน frame ก่อนแสดงผลในหน้าต่าง

```
Video/Webcam → Frame → MediaPipe Pose Landmarker → 33 Landmarks → Draw → Display
```

---

## โครงสร้างไฟล์

```
workshop-7/
├── body-tracking.py           # โค้ดหลัก
├── pose_landmarker_full.task  # Model file ของ MediaPipe (ต้องดาวน์โหลด)
├── videos/
│   └── jumpingjack.mp4        # วิดีโอตัวอย่าง
└── README.md
```

---

## Requirements

| Package | Version | หน้าที่ |
|---|---|---|
| Python | 3.8+ | ภาษาหลัก |
| opencv-python | 4.x | อ่านวิดีโอและแสดงผล |
| mediapipe | 0.10.30+ | ตรวจจับ Pose Landmarks |

---

## การติดตั้ง

### 1. ติดตั้ง Dependencies

```bash
pip install opencv-python mediapipe
```

### 2. ดาวน์โหลด Model File

MediaPipe Pose Landmarker ต้องใช้ไฟล์ `.task` แยกต่างหาก ดาวน์โหลดแล้ววางไว้ใน root ของโปรเจกต์

```bash
curl -L -o pose_landmarker_full.task \
  "https://storage.googleapis.com/mediapipe-models/pose_landmarker/pose_landmarker_full/float16/latest/pose_landmarker_full.task"
```

> Model มี 3 ขนาดให้เลือก: `pose_landmarker_lite` (เร็วสุด), `pose_landmarker_full` (สมดุล), `pose_landmarker_heavy` (แม่นสุด)

---

## วิธีใช้งาน

### รันกับวิดีโอไฟล์ (ค่าเริ่มต้น)

```bash
python3 body-tracking.py
```

โปรแกรมจะเปิดไฟล์ `videos/jumpingjack.mp4` โดยอัตโนมัติ

### เปลี่ยนเป็นกล้องเว็บแคม

เปิดไฟล์ `body-tracking.py` แล้วแก้บรรทัด 6–7:

```python
VIDEO_SOURCE = 0        # เปิดใช้บรรทัดนี้ (0 = กล้องหลัก)
# VIDEO_SOURCE = "videos/jumpingjack.mp4"
```

### ปิดโปรแกรม

กด `Q` ในหน้าต่างแสดงผล หรือรอให้วิดีโอเล่นจบ

---

## อธิบายโค้ด

### ค่าคงที่ `POSE_CONNECTIONS`

```python
POSE_CONNECTIONS = [
    (0,1),(1,2),(2,3),(3,7), ...
]
```

รายการ tuple ของ index คู่ที่ต้องเชื่อมด้วยเส้น เช่น `(11, 13)` คือไหล่ซ้าย → ข้อศอกซ้าย โดย MediaPipe กำหนด landmark 33 จุด ดังนี้:

| Index | ตำแหน่ง | Index | ตำแหน่ง |
|---|---|---|---|
| 0 | จมูก | 11 | ไหล่ซ้าย |
| 1–4 | ตา/หู ซ้าย | 12 | ไหล่ขวา |
| 5–8 | ตา/หู ขวา | 13–14 | ข้อศอก |
| 9–10 | ปาก | 15–16 | ข้อมือ |
| 23–24 | สะโพก | 25–26 | เข่า |
| 27–28 | ข้อเท้า | 29–32 | ส้นเท้า/ปลายเท้า |

### ฟังก์ชัน `draw_landmarks(frame, landmarks)`

วาด skeleton ลงบน frame โดย:
1. แปลงค่า normalized (0.0–1.0) ของแต่ละ landmark ให้เป็น pixel coordinate
2. วาดเส้นสีเขียว `(0, 255, 0)` เชื่อมแต่ละคู่ใน `POSE_CONNECTIONS`
3. วาดวงกลมสีแดง `(0, 0, 255)` ที่จุด landmark แต่ละจุด

### ฟังก์ชัน `main()`

| ขั้นตอน | คำอธิบาย |
|---|---|
| เปิด VideoCapture | รับ input จากไฟล์วิดีโอหรือกล้อง |
| สร้าง PoseLandmarker | โหลด model `.task` พร้อม confidence threshold |
| Loop อ่าน frame | อ่านทีละ frame จนหมดวิดีโอหรือกด Q |
| Flip frame | กลับซ้าย-ขวา (`cv2.flip(..., 1)`) เพื่อให้เป็น mirror effect |
| แปลงสี BGR → RGB | MediaPipe ต้องการ RGB, OpenCV ให้ BGR |
| ตรวจจับ Pose | `landmarker.detect()` คืน landmarks 33 จุด |
| วาดและแสดงผล | `draw_landmarks()` + `cv2.imshow()` |

---

## พารามิเตอร์ที่ปรับได้

ใน `body-tracking.py` บรรทัด 37–42:

```python
options = vision.PoseLandmarkerOptions(
    base_options=base_options,
    output_segmentation_masks=False,   # True = สร้าง mask ของร่างกาย (ช้าลง)
    min_pose_detection_confidence=0.5, # ค่าความมั่นใจขั้นต่ำในการตรวจจับ (0.0–1.0)
    min_tracking_confidence=0.5,       # ค่าความมั่นใจขั้นต่ำในการ tracking (0.0–1.0)
)
```

- **เพิ่ม confidence** → แม่นขึ้น แต่อาจพลาดเมื่อมีการบดบัง
- **ลด confidence** → ตรวจจับได้มากขึ้น แต่อาจเกิด false positive

---

## ปัญหาที่พบบ่อย

| ข้อความ Error | สาเหตุ | วิธีแก้ |
|---|---|---|
| `Cannot open video source` | ไม่พบไฟล์วิดีโอหรือกล้อง | ตรวจสอบ path หรือเปลี่ยน `VIDEO_SOURCE` |
| `module 'mediapipe' has no attribute 'solutions'` | mediapipe เวอร์ชัน 0.10+ เปลี่ยน API | ใช้โค้ดเวอร์ชันนี้ที่ใช้ `mediapipe.tasks` แล้ว |
| `FileNotFoundError: pose_landmarker_full.task` | ไม่มี model file | ดาวน์โหลดตามขั้นตอนด้านบน |
| หน้าต่างไม่เปิด (macOS) | ปัญหา display permission | รันจาก Terminal ปกติ ไม่ใช่ IDE terminal |
