# Sticky Voice Keep-Alive

Multi-token Discord selfbot — mỗi token gắn cứng **một** voice channel và giữ kết nối 24/7 với độ ổn định cao.

**Ưu tiên:** Ổn định > Hiệu suất tài nguyên > Tính năng.

---

## Tính năng chính

- Sticky tuyệt đối: mỗi token chỉ join đúng 1 channel đã map trong `tokens.txt`
- Isolation hoàn toàn: 1 token lỗi / ban / disconnect **không** ảnh hưởng token khác
- Join lần lượt + random delay + semaphore (không spam gateway)
- Silent Opus packets định kỳ (nhẹ CPU)
- Exponential backoff + quarantine khi fail liên tiếp
- Token invalid (401/403) → đánh dấu **DEAD**, không retry
- Persistence session (resume ưu tiên)
- Graceful shutdown (SIGINT / SIGTERM)
- Status summary định kỳ

---

## Yêu cầu

- Python **3.11+**
- Hệ thống có `libopus` (Linux) hoặc Opus DLL (Windows)
- VPS khuyến nghị: 2 vCPU / 2 GB RAM cho 15–30 token

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install -y python3.11 python3.11-venv libopus0 libffi-dev
```

---

## Cài đặt

```bash
git clone <repo> sticky-voice   # hoặc giải nén
cd sticky-voice

python3.11 -m venv .venv
source .venv/bin/activate      # Windows: .venv\Scripts\activate

pip install -U pip
pip install -r requirements.txt
```

---

## Cấu hình

### 1. `config.yaml`

```yaml
guild_id: "123456789012345678"   # ID server
join_delay_min: 5
join_delay_max: 12
max_concurrent_joins: 6
enable_empty_opus: true
empty_opus_interval: 8
log_level: INFO
quarantine_minutes: 45
max_fail_before_quarantine: 6
status_interval: 60
backoff_base: 2
backoff_max: 90
```

### 2. `tokens.txt`

Mỗi dòng:

```
token:channel_id
```

Ví dụ:

```
MTAyNxxxxxxxxxxxxxxx.Gxxxxx.xxxxxxxxxxxxxxxxxxxxxxxx:123456789012345678
OTxxxxxxxxxxxxxxxxxx.Gxxxxx.xxxxxxxxxxxxxxxxxxxxxxxx:987654321098765432
```

- Dòng trống và dòng bắt đầu bằng `#` bị bỏ qua
- Token trùng → chỉ lấy dòng đầu, cảnh báo các dòng sau
- Dòng sai định dạng → log lỗi + bỏ qua, **không crash**

### Lấy token (tự chịu trách nhiệm)

1. Mở Discord web/desktop → DevTools (F12) → Network / Application
2. Hoặc dùng extension lấy token user (chỉ dùng token của chính bạn)
3. **Không** share token. Token = full quyền account.

---

## Chạy thử

```bash
source .venv/bin/activate
python main.py
```

Log mẫu:

```
2026-11-10 18:00:01 | INFO    | sticky | sticky-voice starting
2026-11-10 18:00:01 | INFO    | sticky.tokens | loaded 3 unique token(s)
2026-11-10 18:00:05 | INFO    | sticky.worker | [MTxxxx…] joining channel 123…
2026-11-10 18:00:07 | INFO    | sticky.worker | [MTxxxx…] joined sticky
```

Dừng sạch: `Ctrl+C` hoặc `kill -TERM <pid>`

---

## Chạy 24/7

### systemd

```ini
# /etc/systemd/system/sticky-voice.service
[Unit]
Description=Sticky Voice Keep-Alive
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/sticky-voice
Environment=PATH=/home/ubuntu/sticky-voice/.venv/bin
ExecStart=/home/ubuntu/sticky-voice/.venv/bin/python main.py
Restart=always
RestartSec=10
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now sticky-voice
sudo journalctl -u sticky-voice -f
```

### pm2

```bash
npm i -g pm2
pm2 start main.py --name sticky-voice --interpreter .venv/bin/python
pm2 save
pm2 startup
```

---

## Khuyến nghị vận hành

| Số token | RAM ước tính | Ghi chú |
|----------|--------------|---------|
| 1–10     | ~400–800 MB  | ổn định cao |
| 15–30    | ~1.2–2 GB    | VPS 2vCPU/2GB |
| >40      | cân nhắc tách process / máy | gateway pressure |

- Mỗi token = 1 Gateway WS + 1 Voice WS → đừng nhồi quá nhiều trên 1 IP
- Dùng proxy riêng theo token nếu scale lớn (tính năng proxy để mở rộng sau)
- `max_concurrent_joins` giữ thấp (4–8) để tránh rate-limit
- Token chết → xóa khỏi `tokens.txt` rồi restart

---

## Xử lý sự cố

| Hiện tượng | Nguyên nhân thường gặp | Cách xử lý |
|------------|------------------------|------------|
| Token DEAD ngay | Token invalid / reset | Lấy token mới |
| Quarantine liên tục | Channel không tồn tại / thiếu quyền | Kiểm tra channel_id + role |
| Không join được | guild_id sai / không cùng server | Sửa `config.yaml` |
| CPU cao | empty_opus_interval quá thấp | Tăng lên 10–15s |
| Disconnect hàng loạt | IP bị flag / network | Đổi IP / giảm số token |
| OpusNotLoaded | Thiếu libopus | `apt install libopus0` |

---

## Cấu trúc thư mục

```
sticky-voice/
├── main.py
├── config.yaml
├── tokens.txt
├── requirements.txt
├── README.md
├── core/
│   ├── config.py
│   ├── logger.py
│   ├── token_manager.py
│   └── orchestrator.py
├── worker/
│   ├── voice_worker.py
│   └── connection.py
└── utils/
    ├── backoff.py
    └── persistence.py
```

---

## Lưu ý pháp lý

Automating user accounts vi phạm Discord Terms of Service.  
Dự án này chỉ mang tính nghiên cứu / kỹ thuật.  
Bạn tự chịu mọi rủi ro khi sử dụng (ban account, v.v.).

---

absolute olliE — ship clean.
