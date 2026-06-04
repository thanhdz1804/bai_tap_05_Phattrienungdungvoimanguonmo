# bai_tap_05_Phattrienungdungvoimanguonmo

---

## PHẦN 1: LÝ THUYẾT

### 1. Docker là gì?

Docker là một nền tảng mã nguồn mở cho phép các nhà phát triển tự động hóa việc triển khai, quản lý và chạy các ứng dụng bên trong các môi trường biệt lập được gọi là **Container**.

Khác với ảo hóa truyền thống (Virtual Machine - cần một hệ điều hành khách đầy đủ), Docker Container chia sẻ chung nhân (kernel) của hệ điều hành máy chủ (Host OS). Nhờ vậy, container cực kỳ nhẹ, khởi động trong vài giây và tiêu tốn rất ít tài nguyên.

---

### 2. Các keyword được sử dụng trong docker-compose.yml

`docker-compose.yml` là file cấu hình định dạng YAML dùng để định nghĩa và vận hành các ứng dụng Docker đa container (multi-container).

#### A. Keyword cấp cao nhất (Top-level)

| Keyword | Ý nghĩa |
|---|---|
| `version` | Định nghĩa phiên bản cấu hình của Docker Compose (Ví dụ: `'3.8'`) |
| `services` | Định nghĩa các container cấu thành nên ứng dụng |
| `networks` | Định nghĩa các mạng ảo để các container giao tiếp với nhau |
| `volumes` | Định nghĩa các vùng lưu trữ dữ liệu bền vững (persistent data), không bị mất khi container bị xóa |

#### B. Keyword cấu hình một Service (Container)

| Keyword | Ý nghĩa | Ví dụ trong bài thực hành |
|---|---|---|
| `image` | Tên Docker Image sẽ dùng để tạo container | `nodered/node-red:latest`, `mariadb:10.6`, `influxdb:2.7`, `grafana/grafana:latest`, `nginx:alpine` |
| `container_name` | Đặt tên cố định cho container | `nodered`, `mariadb`, `influxdb`, `grafana`, `nginx`, `flask-api` |
| `restart` | Chính sách tự khởi động lại container | `always` — đảm bảo service luôn chạy khi hệ thống reboot |
| `environment` | Truyền biến môi trường vào container | `MYSQL_ROOT_PASSWORD`, `INFLUXDB_ADMIN_TOKEN`, ... |
| `ports` | Ánh xạ cổng `Host:Container` | `"1880:1880"` (Node-RED), `"3000:3000"` (Grafana), `"80:80"` (Nginx) |
| `volumes` | Gắn thư mục/volume để lưu dữ liệu bền vững | `db_data:/var/lib/mysql`, `./html:/usr/share/nginx/html` |
| `networks` | Chỉ định mạng ảo mà container tham gia | `app_net` — tất cả service dùng chung 1 mạng để liên lạc |
| `depends_on` | Xác định thứ tự khởi động (service này cần service kia chạy trước) | `flask-api` phụ thuộc `mariadb`; `grafana` phụ thuộc `influxdb` |
| `build` | Build image từ Dockerfile thay vì dùng image có sẵn | Dùng cho `flask-api` (tự viết code) |

#### C. Ví dụ file docker-compose.yml của bài thực hành

```yaml
version: '3.8'

services:

  # 1. Node-RED: lấy dữ liệu thực tế và điều phối toàn bộ luồng
  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: always
    ports:
      - "1880:1880"
    volumes:
      - nodered_data:/data
    networks:
      - app_net

  # 2. MariaDB: lưu giá trị tức thời
  mariadb:
    image: mariadb:10.6
    container_name: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: realtime_db
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - app_net

  # 3. InfluxDB: lưu dữ liệu lịch sử (time-series)
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: always
    ports:
      - "8086:8086"
    volumes:
      - influx_data:/var/lib/influxdb2
    networks:
      - app_net

  # 4. Grafana: trực quan hoá dữ liệu lịch sử
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    depends_on:
      - influxdb
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - app_net

  # 5. Flask API: trả dữ liệu tức thời từ MariaDB cho Frontend
  flask-api:
    build: ./flask-api
    container_name: flask-api
    restart: always
    ports:
      - "5000:5000"
    depends_on:
      - mariadb
    networks:
      - app_net

  # 6. Nginx: webserver phục vụ trang HTML frontend
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - flask-api
      - grafana
    networks:
      - app_net

volumes:
  nodered_data:
  db_data:
  influx_data:
  grafana_data:

networks:
  app_net:
    driver: bridge
```

---

### 3. Ưu điểm khi triển khai ứng dụng bằng Docker

| Ưu điểm | Giải thích |
|---|---|
| **Tính nhất quán** (Consistency) | Giải quyết triệt để lỗi *"Chạy tốt trên máy tôi nhưng lỗi trên server"*. Image đóng gói mọi thứ (code, runtime, thư viện, cấu hình) |
| **Trọng lượng nhẹ & Hiệu năng cao** | Container khởi động tính bằng mili-giây, tiêu tốn ít RAM/CPU hơn rất nhiều so với VM |
| **Cô lập môi trường** (Isolation) | Mỗi container độc lập — Node-RED, MariaDB, InfluxDB, Grafana chạy song song không xung đột |
| **Dễ mở rộng & chuyển dịch** (Portability) | Chỉ cần cài Docker, bê toàn bộ hệ thống từ Laptop lên Server vật lý, AWS, Azure bằng vài lệnh |
| **Quản lý hạ tầng bằng Code** (IaC) | Toàn bộ kiến trúc (6 service, network, volume) định nghĩa gọn trong 1 file `docker-compose.yml` |

---

### 4. Quy trình triển khai Offline (Server không có Internet)

Khi máy chủ đích bị ngắt Internet (môi trường Air-gapped), quy trình deploy từ laptop cá nhân gồm 4 bước:

**Bước 1: Đóng gói trên Laptop (có Internet)**

Build và chạy thử nghiệm hệ thống thành công:
```bash
docker compose up -d
docker ps          # xác nhận tất cả container đang Up
docker images      # xem danh sách image đang dùng
```

**Bước 2: Export (Save) các Image ra file nén Tar**

Nén tất cả image của bài thực hành vào 1 file duy nhất:
```bash
docker save -o app_images.tar \
  nodered/node-red:latest \
  mariadb:10.6 \
  influxdb:2.7 \
  grafana/grafana:latest \
  nginx:alpine \
  flask-api:latest
```

**Bước 3: Chuyển file sang Máy chủ Offline**

Dùng USB, ổ cứng di động, hoặc mạng LAN (SFTP/SCP) để sao chép:
- `app_images.tar` — chứa toàn bộ Docker Images
- `docker-compose.yml` — file cấu hình hệ thống
- `./flask-api/` — thư mục source code Flask API
- `./html/` — thư mục frontend HTML/JS/CSS
- `./nginx/` — thư mục cấu hình Nginx

**Bước 4: Import (Load) Image và Chạy ứng dụng trên Máy chủ**

```bash
# Giải nén các Docker Image
docker load -i app_images.tar

# Kiểm tra image đã vào hệ thống chưa
docker images

# Kích hoạt toàn bộ hệ thống
docker compose up -d

# Xác nhận tất cả service đang chạy
docker ps
```

---

## Thực hành áp dụng: APP MONITOR + ALERT DATA REALTIME
## 1. 🐳 DOCKER COMPOSE – KHỞI ĐỘNG SERVICES

### Những Lưu Ý Nằm Lòng Cho Lần Sau (Trước Khi docker-compose up -d)
Để các dự án sau này chạy mượt mà ngay từ lần đầu tiên, trước khi gõ lệnh up, bạn hãy tạo thành thói quen kiểm tra các yếu tố sau:
Luôn kiểm tra dung lượng đĩa trước: Gõ df -h để chắc chắn ổ cứng còn trống tối thiểu vài GB. Nếu thấy phân vùng bị bóp nhỏ (như lỗi 14GB vừa rồi), hãy dùng lệnh lvextend và resize2fs để nới rộng ra ngay từ đầu.
Không để file cấu hình trống: Khi clone hoặc tạo cấu hình có chứa thuộc tính build:, hãy đảm bảo file Dockerfile đã được viết nội dung, không để file trống 0 bytes.
Mẹo tải Image khi mạng yếu: Nếu mạng chập chờn, thay vì gõ thẳng docker-compose up -d, hãy chủ động kéo trước các image nặng bằng lệnh lẻ: docker pull <tên_image>. Khi các khối dữ liệu lớn đã nằm an toàn trên máy, việc chạy lệnh up sẽ không bao giờ lo bị lỗi kết nối nữa.
Kiểm tra DNS của Server: Đảm bảo file /etc/resolv.conf luôn trỏ về các DNS mạnh như 8.8.8.8 hoặc 1.1.1.1 để tránh server bị mất phương hướng khi tìm tên miền của các dịch vụ quốc tế.

### 1.1  Nội dung file `docker-compose.yml` (mở trong editor hoặc terminal `cat docker-compose.yml`)  
```yaml
version: '3.8'

services:

  # 1. Node-RED: lấy dữ liệu thực tế và điều phối toàn bộ luồng
  nodered:
    image: nodered/node-red:latest
    container_name: nodered
    restart: always
    ports:
      - "1880:1880"
    volumes:
      - nodered_data:/data
    networks:
      - app_net

  # 2. MariaDB: lưu giá trị tức thời
  mariadb:
    image: mariadb:10.6
    container_name: mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: realtime_db
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql
    networks:
      - app_net

  # 3. InfluxDB: lưu dữ liệu lịch sử (time-series)
  influxdb:
    image: influxdb:2.7
    container_name: influxdb
    restart: always
    ports:
      - "8086:8086"
    volumes:
      - influx_data:/var/lib/influxdb2
    networks:
      - app_net

  # 4. Grafana: trực quan hoá dữ liệu lịch sử
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    restart: always
    ports:
      - "3000:3000"
    depends_on:
      - influxdb
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - app_net

  # 5. Flask API: trả dữ liệu tức thời từ MariaDB cho Frontend
  flask-api:
    build: ./flask-api
    container_name: flask-api
    restart: always
    ports:
      - "5000:5000"
    depends_on:
      - mariadb
    networks:
      - app_net

  # 6. Nginx: webserver phục vụ trang HTML frontend
  nginx:
    image: nginx:alpine
    container_name: nginx
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./html:/usr/share/nginx/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - flask-api
      - grafana
    networks:
      - app_net

volumes:
  nodered_data:
  db_data:
  influx_data:
  grafana_data:

networks:
  app_net:
    driver: bridge
```

---
### 1.2  Chạy lệnh `docker-compose up -d` — terminal hiển thị các container đang được tạo 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/643acc80-d8aa-4cab-b151-c92116454b09" />

### 1.3  Kết quả lệnh `docker ps` — tất cả container đang **Up** (nodered, mariadb, influxdb, grafana, nginx, flask)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/8a989522-e1c2-4575-993a-a7db6c353d64" />


---

## 2. 🔄 NODE-RED – LẤY DỮ LIỆU THỰC TẾ
### 2.1  Giao diện Node-RED Editor — toàn bộ flow đang chạy (các node kết nối nhau)
<img width="1902" height="1068" alt="image" src="https://github.com/user-attachments/assets/30feeed0-54c6-4af1-8a70-6acc362b9d7b" />

### 2.2  Cấu hình node lấy dữ liệu (HTTP Request / API node) — URL nguồn dữ liệu thực (chứng khoán)
<img width="1913" height="1058" alt="image" src="https://github.com/user-attachments/assets/ef969a2f-88de-4ebf-9a3b-c5e0d1725a5f" />

### 2.3  Debug panel Node-RED — hiển thị dữ liệu thực đang được nhận về (có giá trị số thực tế, timestamp)  
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/289f0734-fa4d-418f-bcce-4c7d56e39791" />

### 2.4  Node-RED đang **chạy liên tục** — inject node trigger theo interval
<img width="1912" height="1080" alt="image" src="https://github.com/user-attachments/assets/d2ccba27-493e-4361-b321-1e305dea190c" />


---

## 3. 🗄️ DATABASE – LƯU TRỮ DỮ LIỆU

### 3a. MariaDB (giá trị tức thời)
#### 3.1  Cấu hình node MariaDB trong Node-RED (host, database, table) 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/72be570d-4ac7-4e18-bfab-8cc4f67499b9" />

#### 3.2 Truy vấn trong MariaDB — `SELECT * FROM <table> ORDER BY id DESC LIMIT 10;` — thấy dữ liệu đã được lưu 
<img width="1887" height="1080" alt="image" src="https://github.com/user-attachments/assets/eff4ae6c-4e52-43d4-b0c4-b9ee16be8011" />

### 3b. InfluxDB (lịch sử)

#### 3.3 Cấu hình node InfluxDB trong Node-RED (bucket, measurement)
<img width="1918" height="1080" alt="image" src="https://github.com/user-attachments/assets/4935fe4a-6558-4862-b907-117472ecf362" />
<img width="1917" height="1063" alt="image" src="https://github.com/user-attachments/assets/24f67951-8afe-4bc7-995e-1209ee221493" />

#### 3.4  Giao diện InfluxDB UI (`:8086`) — Data Explorer hiển thị dữ liệu lịch sử đã được ghi vào 
<img width="1920" height="1076" alt="image" src="https://github.com/user-attachments/assets/86442ad9-5e68-404a-a816-5d4faca9e333" />

---

## 4. 📊 GRAFANA – BIỂU ĐỒ TRỰC QUAN HOÁ
### 4.1 Cấu hình Data Source InfluxDB trong Grafana (Settings → Data Sources — trạng thái **Data source connected**)
<img width="1919" height="1079" alt="image" src="https://github.com/user-attachments/assets/2c74438f-88bf-492a-a17f-f6625e59352f" />

### 4.2 Dashboard Grafana — biểu đồ thể hiện dữ liệu lịch sử theo thời gian (time series chart có dữ liệu thực) 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b6b22943-d8f7-41b2-8217-fafecf1df12d" />

### 4.3 Cấu hình **Alert Rule** trong Grafana — thiết lập ngưỡng ALERT LOW / ALERT HIGH 
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b2f9c074-a8fe-4494-9d65-3084fdb4cfda" />
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/014ea07c-c9c2-4465-b4e9-dfef359a03c5" />


---

## 5. 🌐 FLASK API – BACKEND TRẢ DỮ LIỆU TỨC THỜI

| # | Nội dung cần chụp | Tên file gợi ý |
|---|---|---|
| 5.1 | Nội dung file `app.py` (Flask) — endpoint trả dữ liệu từ MariaDB | `15_flask-app-code.png` |
| 5.2 | Gọi API trực tiếp trên trình duyệt hoặc `curl` — JSON trả về có giá trị tức thời | `16_flask-api-response.png` |

---

## 6. 🖥️ NGINX + FRONTEND WEB

| # | Nội dung cần chụp | Tên file gợi ý |
|---|---|---|
| 6.1 | Nội dung file cấu hình `nginx.conf` (hoặc `default.conf`) | `17_nginx-config.png` |
| 6.2 | Trang web chạy trên trình duyệt (qua Nginx) — hiển thị đầy đủ giao diện | `18_frontend-web-ui.png` |
| 6.3 | Giá trị **tự động cập nhật** trên web — chụp 2 ảnh cách nhau vài giây, thấy số thay đổi (hoặc dùng DevTools → Network thấy AJAX request liên tục) | `19_frontend-auto-update.png` |
| 6.4 | **iframe Grafana** hiển thị biểu đồ nhúng trong trang web | `20_frontend-grafana-iframe.png` |

---

## 7. 🚨 ALERT – PHÁT HIỆN BẤT THƯỜNG

| # | Nội dung cần chụp | Tên file gợi ý |
|---|---|---|
| 7.1 | Node-RED — flow xử lý logic so sánh ngưỡng (switch node / function node kiểm tra ALERT LOW / HIGH) | `21_nodered-alert-logic.png` |
| 7.2 | Cấu hình ngưỡng trong flow (mở node, thấy rõ giá trị ngưỡng A và B) | `22_nodered-threshold-config.png` |

---

## 8. 📲 TELEGRAM BOT – GỬI CẢNH BÁO

| # | Nội dung cần chụp | Tên file gợi ý |
|---|---|---|
| 8.1 | Cấu hình node Telegram trong Node-RED (bot token, chat ID của group) | `23_nodered-telegram-node.png` |
| 8.2 | **Tin nhắn alert thực tế** được gửi vào group Telegram — thấy rõ: tên bot, nội dung tường minh, **giá trị gây alert**, loại alert (LOW/HIGH), timestamp | `24_telegram-alert-message.png` |
| 8.3 | Danh sách thành viên group Telegram — có đủ 3 người (bao gồm ID `1875746636`) | `25_telegram-group-members.png` |

---

## 9. 📦 XUẤT – XOÁ – KHÔI PHỤC CONTAINER

| # | Nội dung cần chụp | Tên file gợi ý |
|---|---|---|
| 9.1 | Lệnh **export** từng container ra file (hoặc script tổng hợp), ví dụ: `docker save ... -o containers.tar` | `26_docker-export-command.png` |
| 9.2 | File nén đã được tạo ra — `ls -lh *.tar` hoặc thấy file trong folder | `27_docker-export-file.png` |
| 9.3 | Lệnh **xoá** tất cả container — `docker compose down` hoặc `docker rm -f $(docker ps -aq)` | `28_docker-remove-containers.png` |
| 9.4 | Xác nhận đã xoá — `docker ps -a` — **không còn container nào** | `29_docker-ps-empty.png` |
| 9.5 | Lệnh **load lại** từ file nén — `docker load -i ...` hoặc `docker compose up` sau khi load | `30_docker-load-command.png` |
| 9.6 | Sau khi khôi phục — `docker ps` lại thấy **đầy đủ các container đang Up** | `31_docker-restored-running.png` |

---
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/66e425db-b755-4a7c-8a60-4e2284637eaa" />



### + QUAN SÁT DỮ LIỆU LỊCH SỬ => GIÁ TRỊ BẤT THƯỜNG
       (VD MIỀN A..B: OK, DƯỚI A: ALERT LOW, TRÊN B: ALERT HIGH)
### + nodered: kết hợp bot Telegram
       khi dữ liệu not OK, thì gửi tin nhắn từ bot => group trên telegram
       group đã add bot vào: (nhóm đã có 2 người), add thêm 1875746636 thành 3 người
       mỗi khi bot gửi dữ liệu vào nhóm: mọi member of group đều nhận đc
       nội dung alert: tường minh, có value gây alert

     xuất tất cả các container ra file nén.
     xoá mọi container đang chạy
     load lại các container  từ file nén để khôi phục các container đã xoá
=========
quá trình làm: chụp ảnh lại, mô tả cho ảnh
  lưu vào trong github => paste link access public của repo: vào file excel online
