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
## PHẦN 2: THỰC HÀNH

### Kiến trúc tổng thể hệ thống

```
Nguồn dữ liệu thực tế (API giá vàng / thời tiết / chứng khoán)
        │
        ▼
  [Node-RED :1880]  ──── lấy dữ liệu theo interval ────┐
        │                                               │
        ├──── lưu tức thời ──► [MariaDB :3306]          │
        │                           │                   │
        ├──── lưu lịch sử ──► [InfluxDB :8086]          │
        │                           │                   │
        └──── alert logic ──► [Telegram Bot] ◄──────────┘
                                    
  [Flask API :5000] ◄──── query tức thời ──── MariaDB
        │
        ▼
  [Nginx :80] ──── phục vụ Frontend HTML/JS/CSS
        │
        ├──── AJAX/Socket ──► Flask API ──► giá trị tức thời
        └──── iframe ──────► [Grafana :3000] ──► biểu đồ lịch sử
```

---
## 1. Tạo thư mục project
Bước 1
-Tạo thư mục:
mkdir realtime-monitor
cd realtime-monitor

mkdir flask_api
mkdir web
mkdir mariadb_data
mkdir influxdb_data
mkdir grafana_data
mkdir nodered_data
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/788998eb-e409-4062-8dc0-b3d57e911137" />
## 2. Tạo docker-compose.yml
tạo file:    nano docker-compose.yml
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
<img width="1916" height="1080" alt="image" src="https://github.com/user-attachments/assets/feb7b0e0-5420-43fd-be7f-449bbe655211" />

### 3. Tạo Flask API
#### Tạo Dockerfile: nano flask_api/Dockerfile
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c6e7c8f1-cfe9-410e-8309-36022eccccad" />

#### Tạo file: nano flask_api/app.py
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/8cebee7f-86fc-4b14-b77f-805e64df1df6" />

### 4. Tạo giao diện web
#### Tạo file:   nano web/index.html
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/9534be60-949f-419f-9a11-c8a2d92812a9" />

### 5. Chạy Docker Compose
Chạy toàn bộ hệ thống:  docker-compose up -d --build

Kiểm tra container:   docker ps
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/f6fe1921-ae95-4f84-843a-2a1eabafd3eb" />

### 6. Tạo database MariaDB bằng DBeaver
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/77566e8b-9e35-4030-928c-380075da0b78" />

#### Tạo bảng
<img width="1912" height="1080" alt="image" src="https://github.com/user-attachments/assets/69078701-bf2c-49cd-9c33-54f848ab006f" />
### 7. 

