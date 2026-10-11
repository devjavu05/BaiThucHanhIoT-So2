# BTH2 — Thu thập, Lưu trữ và Tiền xử lý Dữ liệu IoT

**Học phần:** IoT và Ứng dụng (INT14149)
**Sinh viên:** AnhAnh
**Repo:** _đẩy lên GitHub của bạn_

---

## 1. Kiến trúc hệ thống

```
 ┌──────────┐   MQTT (JSON)   ┌─────────────┐    HTTP     ┌───────────────┐
 │  ESP32   │───────────────▶ │  Mosquitto  │ ───────────▶│  Collector    │
 │ DHT22+LDR│  iot/<id>/tele  │   broker    │             │ (Python)      │
 └──────────┘                 └─────────────┘             └──────┬────────┘
                                                                 │ influxdb-client
                                                                 ▼
                                                           ┌───────────────┐
                                                           │  InfluxDB 2.x │
                                                           │ bucket iot_raw│
                                                           └──────┬────────┘
                                                                  │
                                                 ┌────────────────┴───────────────┐
                                                 ▼                                ▼
                                          preprocess.py                   dashboard/app.py
                                        (clean/outlier/                   (Streamlit)
                                         resample/features)
                                                 │
                                                 ▼
                                       bucket iot_processed
```

## 2. Cấu trúc dự án

```
iot-lab2/
├── firmware/
│   ├── esp32_sensor.ino       # Firmware Arduino cho ESP32 (DHT22 + LDR) — board thật
│   ├── esp32_sim.py           # Trình giả lập Python khi không có board
│   └── wokwi/                 # Project Wokwi (sketch.ino + diagram.json + README)
│       ├── sketch.ino
│       ├── diagram.json
│       ├── libraries.txt
│       ├── wokwi.toml
│       └── README.md
├── collector/
│   └── mqtt_to_influx.py      # Subscribe MQTT → validate → ghi InfluxDB
├── preprocessing/
│   └── preprocess.py          # Làm sạch, outlier, resample, feature, chuẩn hoá
├── dashboard/
│   └── app.py                 # Streamlit dashboard
├── config/
│   ├── config.yaml            # Cấu hình chung (broker localhost)
│   ├── config.wokwi.yaml      # Cấu hình khi dùng Wokwi (broker public)
│   └── docker-compose.yml     # Mosquitto + InfluxDB + Grafana
├── mosquitto/config/
│   └── mosquitto.conf
├── scripts/
│   └── init_influxdb.sh       # Tạo bucket iot_processed + retention
├── docs/
│   └── BaoCao_BTH2.docx       # Báo cáo 4–6 trang
├── requirements.txt
└── README.md
```

## 3. Chuẩn bị môi trường

### 3.1 Phần cứng
Bản thực tế của AnhAnh dùng ESP32 giả lập trên Wokwi với bộ cảm biến:
- **ESP32 DevKit v1**
- **DHT22** (data → GPIO15, VCC 3V3)
- **HC-SR04** siêu âm (TRIG → GPIO5, ECHO → GPIO18, VCC → VIN)
- **LED** xanh + điện trở 220 Ω trên GPIO12 làm đèn báo publish

(Nếu dùng board thật có LDR thay cho HC-SR04, chỉ cần đổi `distance_cm`
thành `light` trong `FIELDS` của `preprocessing/preprocess.py` và trong
truy vấn Flux của dashboard — pipeline còn lại không đổi.)

### 3.2 Phần mềm
```bash
# Clone repo rồi:
cd iot-lab2
python -m venv .venv && source .venv/bin/activate    # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 3.3 Khởi tạo hạ tầng Docker
```bash
docker compose -f config/docker-compose.yml up -d
bash scripts/init_influxdb.sh
```
- InfluxDB UI: <http://localhost:8086> (user `admin` / pass `adminpassword`)
- Grafana  : <http://localhost:3000> (user `admin` / pass `admin`)
- Mosquitto: `localhost:1883`

> Token InfluxDB mặc định: `my-super-secret-admin-token` — đã có sẵn trong
> `config/config.yaml`. **Đổi** khi deploy thật.

## 4. Chạy pipeline

### 4.1 Thu thập dữ liệu — chọn 1 trong 3 cách

**Cách A — Giả lập trên Wokwi (ưu tiên nếu không có board):**
Xem hướng dẫn chi tiết ở [`firmware/wokwi/README.md`](firmware/wokwi/README.md).
Tóm tắt:
1. New Project → ESP32 (Arduino) trên <https://wokwi.com>.
2. Dán `firmware/wokwi/sketch.ino` + `firmware/wokwi/diagram.json`, cài 4 thư viện trong `libraries.txt`.
3. ▶ Start Simulation — mạch tự publish lên `broker.hivemq.com`.
4. Ở local, chạy collector với config Wokwi:
   ```bash
   CONFIG_PATH=config/config.wokwi.yaml python collector/mqtt_to_influx.py
   ```
⚠️ Nhớ đổi `TOPIC_PREFIX` trong `sketch.ino` cho khớp `topic` trong `config.wokwi.yaml` (prefix riêng để không đụng người khác dùng chung broker public).

**Cách B — Có board ESP32 thật:**
1. Mở `firmware/esp32_sensor.ino` trong Arduino IDE.
2. Chỉnh `WIFI_SSID`, `WIFI_PASSWORD`, `MQTT_HOST` cho đúng IP máy chạy
   Mosquitto (ví dụ `192.168.1.100`).
3. Cài thư viện: `PubSubClient`, `DHT sensor library`, `Adafruit Unified Sensor`,
   `ArduinoJson` (v6.x).
4. Chọn board **ESP32 Dev Module**, upload.
5. Mở Serial Monitor 115200 baud để kiểm tra log.

**Cách C — Giả lập Python trên máy (không cần Wokwi/board):**
```bash
python firmware/esp32_sim.py --host localhost --device esp32-sim-01 --interval 2
```

### 4.2 Chạy collector
```bash
python collector/mqtt_to_influx.py
# log định kỳ: metrics: in=... written=... invalid=... dup=...
```

### 4.3 Chạy tiền xử lý
```bash
# Tiền xử lý 1 giờ gần nhất, cửa sổ 1 phút
python preprocessing/preprocess.py --since -1h --window 1min

# Xuất CSV để kiểm tra tay / đưa vào báo cáo
python preprocessing/preprocess.py --since -6h --window 5min \
    --export-csv exports/processed.csv
```
Chạy định kỳ với `cron` mỗi 5 phút:
```
*/5 * * * * cd /path/to/iot-lab2 && .venv/bin/python preprocessing/preprocess.py --since -10m --window 1min >> logs/preprocess.log 2>&1
```

### 4.4 Dashboard
```bash
streamlit run dashboard/app.py
# Mở http://localhost:8501
```

## 5. Định dạng JSON thiết bị publish
Topic: `iot/<device_id>/telemetry`
```json
{
  "device_id": "esp32-01",
  "location": "lab-ptit",
  "timestamp": "2026-10-11T08:12:34+07:00",
  "seq": 42,
  "uptime_s": 325,
  "rssi": -58,
  "data": {
    "temperature": 28.7,
    "humidity": 65.4,
    "distance_cm": 152.3
  },
  "unit_temperature": "C",
  "unit_humidity": "%",
  "unit_distance": "cm"
}
```
Thiết bị cũng publish `iot/<id>/status` dạng `{"status":"online"|"offline"}`
(retained + LWT) để dashboard biết trạng thái.

## 6. Schema InfluxDB

### bucket `iot_raw` — retention 30 ngày
| measurement   | tags                        | fields                                                                        |
|---------------|-----------------------------|-------------------------------------------------------------------------------|
| `sensor_raw`  | `device_id`, `location`, `q_*` | `temperature`, `humidity`, `distance_cm`, `seq`, `rssi`, `e2e_latency_ms` |

### bucket `iot_processed` — retention 90 ngày
| measurement        | tags        | fields                                                                                                                                 |
|--------------------|-------------|----------------------------------------------------------------------------------------------------------------------------------------|
| `sensor_processed` | `device_id` | `temperature`, `humidity`, `distance_cm`, `*_rollmean5`, `*_rollstd5`, `*_delta`, `*_roc`, `*_minmax`, `*_zscore`, `*_is_outlier`, `dew_point` |

Chống trùng lặp: collector có LRU `(device_id, seq)`; mất mát được phát hiện qua
giá trị `seq` không liên tục (có thể viết Flux để alert).

## 7. Troubleshooting

| Lỗi | Nguyên nhân thường gặp | Cách xử lý |
|-----|------------------------|------------|
| Collector in=0 | Sai `MQTT_HOST`, firewall, `topic` khớp sai | kiểm tra `mosquitto_sub -t 'iot/#' -v` |
| 401/403 từ Influx | Token sai / hết hạn | tạo lại token trong UI, cập nhật `config.yaml` |
| ESP32 reset liên tục | WDT do publish quá chậm | giảm `PUBLISH_INTERVAL_MS` xuống ≥ 1000 ms |
| Dashboard trống | Chưa tạo bucket `iot_processed` | chạy `scripts/init_influxdb.sh` |
| Giá trị âm/ngoài khoảng | DHT22 lỗi, dây rời | preprocess đã tag + loại outlier; kiểm tra `q_*` tag |

## 8. Mở rộng (tuỳ chọn)
- **TLS**: bật port 8883 Mosquitto, cấp cert tự ký, chỉnh `WiFiClientSecure` trên ESP32.
- **Edge processing**: lọc + rolling mean ngay trên ESP32 để giảm băng thông.
- **So sánh DB**: ghi song song sang MongoDB Time-Series và đo ingest throughput,
  dung lượng, latency truy vấn 1h / 24h.

---
_© 2026 AnhAnh — PTIT · IoT và Ứng dụng (INT14149)_
#   B a i T h u c H a n h I o T - S o 2  
 #   B a i T h u c H a n h I o T - S o 2  
 