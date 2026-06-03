# bai_tap_05_Phattrienungdungvoimanguonmo

## lý thuyết: 
### 1. docker là gì?
Docker là một nền tảng mã nguồn mở cho phép các nhà phát triển tự động hóa việc triển khai, quản lý và chạy các ứng dụng bên trong các môi trường biệt lập được gọi là Container.
Khác với ảo hóa truyền thống (Virtual Machine - cần một hệ điều hành khách đầy đủ), Docker Container chia sẻ chung nhân (kernel) của hệ điều hành máy chủ (Host OS). Nhờ vậy, container cực kỳ nhẹ, khởi động trong vài giây và tiêu tốn rất ít tài nguyên.      


### 2. các keyword được sử dụng trong docker-compose.yml
docker-compose.yml là file cấu hình định dạng YAML dùng để định nghĩa và vận hành các ứng dụng Docker đa container (multi-container).
#### A. Keyword cấp cao nhất (Top-level)
-version: Định nghĩa phiên bản cấu hình của Docker Compose (Ví dụ: '3.8').
-services: Định nghĩa các container cấu thành nên ứng dụng.
-networks: Định nghĩa các mạng ảo để các container giao tiếp với nhau.
-volumes: Định nghĩa các vùng lưu trữ dữ liệu bền vững (persistent data), không bị mất khi container bị xóa.
#### B. Keyword cấu hình một Service (Container)
<img width="1359" height="292" alt="image" src="https://github.com/user-attachments/assets/f163b9b8-7443-48b3-a06d-673382f1b234" />

#### C. Ví dụ một file docker-compose.yml hoàn chỉnh:
```yaml
version: '3.8'

services:
  db:
    image: mariadb:10.6
    container_name: my_mariadb
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

  web:
    image: nginx:alpine
    container_name: my_webserver
    ports:
      - "80:80"
    depends_on:
      - db
    networks:
      - app_net

volumes:
  db_data:

networks:
  app_net:
    driver: bridge
```
### 3. Ưu điểm khi triển khai ứng dụng bằng Docker
-Tính nhất quán (Consistency): Giải quyết triệt để lỗi "Chạy tốt trên máy tôi nhưng lỗi trên máy server". Image đóng gói mọi thứ (code, runtime, thư viện, cấu hình), đảm bảo chạy giống nhau ở mọi môi trường.
-Trọng lượng nhẹ & Hiệu năng cao: Container khởi động tính bằng mili-giây, tiêu tốn ít RAM/CPU hơn rất nhiều so với Máy ảo (VM).
-Cô lập môi trường (Isolation): Mỗi container là một môi trường độc lập. Bạn có thể chạy ứng dụng cần Node.js v14 và ứng dụng cần Node.js v20 trên cùng một máy chủ mà không sợ xung đột.
-Dễ dàng mở rộng và chuyển dịch (Portability): Chỉ cần cài Docker, bạn có thể bê toàn bộ hệ thống từ Laptop lên Server vật lý, AWS, Azure... bằng một vài câu lệnh.
-Quản lý hạ tầng bằng Code (IaC): Toàn bộ kiến trúc hệ thống (DB, Web, Network, Volume) được định nghĩa gọn gàng trong file docker-compose.yml.
 ### 4. Quy trình triển khai Offline (Server thật không có Internet)
Khi máy chủ đích bị ngắt Internet (môi trường Air-gapped), quy trình deploy từ laptop cá nhân qua server gồm 4 bước:
-Bước 1: Đóng gói trên Laptop (Có Internet)
Build và chạy thử nghiệm hệ thống bằng docker compose up -d thành công.
Xác định danh sách các image đang dùng bằng lệnh: docker ps hoặc docker images.
-Bước 2: Export (Save) các Image ra file nén Tar
Sử dụng lệnh docker save để nén các image thành file .tar.
Mẹo: Bạn có thể nén nhiều image vào 1 file duy nhất:
```
docker save -o app_images.tar nodered/node-red:latest mariadb:10.6 influxdb:2.7 grafana/grafana:latest nginx:alpine my-flask-api:latest
```
-Bước 3: Chuyển file sang Máy chủ Offline
Dùng USB, ổ cứng di động, hoặc mạng LAN (qua giao thức SFTP/SCP) để sao chép file app_images.tar và file docker-compose.yml sang máy chủ đích.
-Bước 4: Import (Load) Image và Chạy ứng dụng trên Máy chủ
Trên máy chủ, giải nén các Docker Image bằng lệnh:
```
docker load -i app_images.tar
```
Kiểm tra lại xem image đã vào hệ thống chưa: docker images.
Di chuyển đến thư mục chứa file docker-compose.yml và kích hoạt hệ thống:
```
docker compose up -d
```
## Thực hành áp dụng: APP MONITOR + ALERT DATA REALTIME
### sử dụng docker compose có nhiều serivce 
    và các thành phần cần thiết để tạo thành ứng dụng:
     + nodered liên tục lấy dữ liệu từ nguồn nào đó (chứng khoán, thời tiết, giá vàng,...)
       nguồn thực tế, số liệu luôn động sau thời gian ngắn
     + nodered lưu trữ dữ liệu vào 2 database: mariadb để lưu giá trị tức thời
       lưu lịch sử vào influxdb
     + sử dụng grafana để trực quan hoá dữ liệu: vẽ biểu đồ
     + sử dụng nginx để làm webserver
       chạy 1 trang web html+js+css làm front-end
       js: lấy dữ liệu tức thời trong mariadb qua (ajax | socket) 
           gọi api (api tự build bằng Flask giống bt1)
           api trả về giá trị tức thời trong mariadb
           hiển thị lên web, auto hiển thị số mới khi thay đổi
       sử dụng iframe để gọi grafana
       hiển thị biểu đồ dữ liệu lịch sử của thông số đã lưu
  <img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/685c37b0-8e5a-466e-a49b-769209084066" />
### code docker trong bài

```yaml
version: '3.8'

services:
  # 1. MariaDB: Lưu trữ dữ liệu tức thời (Current value)
  mariadb:
    image: mariadb:10.6
    container_name: monitor_mariadb
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_DATABASE: monitoring_db
      MYSQL_USER: monitor_user
      MYSQL_PASSWORD: monitor_password
    ports:
      - "3306:3306"
    volumes:
      - mariadb_data:/var/lib/mysql
    networks:
      - monitor_net

  # 2. InfluxDB: Lưu trữ dữ liệu chuỗi thời gian (History)
  influxdb:
    image: influxdb:2.7
    container_name: monitor_influxdb
    restart: always
    ports:
      - "8086:8086"
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=admin_password123
      - DOCKER_INFLUXDB_INIT_ORG=my_org
      - DOCKER_INFLUXDB_INIT_BUCKET=history_bucket
    volumes:
      - influxdb_data:/var/lib/influxdb2
    networks:
      - monitor_net

  # 3. Node-RED: Lấy data source, phân tích ngưỡng, lưu DB, bắn Telegram Alert
  nodered:
    image: nodered/node-red:latest
    container_name: monitor_nodered
    restart: always
    ports:
      - "1880:1880"
    volumes:
      - nodered_data:/data
    depends_on:
      - mariadb
      - influxdb
    networks:
      - monitor_net

  # 4. Flask API: Lấy dữ liệu từ MariaDB trả về dạng JSON cho Frontend
  flask-api:
    build: ./flask-api
    container_name: monitor_flask_api
    restart: always
    ports:
      - "5000:5000"
    depends_on:
      - mariadb
    networks:
      - monitor_net

  # 5. Grafana: Vẽ biểu đồ lịch sử từ InfluxDB
  grafana:
    image: grafana/grafana:latest
    container_name: monitor_grafana
    restart: always
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ALLOW_EMBEDDING=true # Cho phép nhúng iframe vào trang web khác
      - GF_AUTH_ANONYMOUS_ENABLED=true   # Cho phép xem biểu đồ không cần đăng nhập (nếu cần)
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - influxdb
    networks:
      - monitor_net

  # 6. Nginx Webserver: Chạy trang Front-end (HTML/JS/CSS) nhúng Iframe Grafana
  nginx:
    image: nginx:alpine
    container_name: monitor_webserver
    restart: always
    ports:
      - "80:80"
    volumes:
      - ./nginx/html:/usr/share/nginx/html
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
    depends_on:
      - flask-api
      - grafana
    networks:
      - monitor_net

volumes:
  mariadb_data:
  influxdb_data:
  nodered_data:
  grafana_data:

networks:
  monitor_net:
    driver: bridge
```

     + QUAN SÁT DỮ LIỆU LỊCH SỬ => GIÁ TRỊ BẤT THƯỜNG
       (VD MIỀN A..B: OK, DƯỚI A: ALERT LOW, TRÊN B: ALERT HIGH)
     + nodered: kết hợp bot Telegram
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
