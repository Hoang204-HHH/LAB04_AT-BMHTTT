# LAB 4 – Khảo sát và đánh giá bề mặt mạng bằng Nmap

Báo cáo thực hành môn **An toàn hệ thống thông tin** – Bộ môn An toàn Thông tin.

## Thông tin sinh viên

| Thông tin | Nội dung |
| :-- | :-- |
| Họ và tên | Huỳnh Hữu Hoàng |
| MSSV | 1150080095 |
| Lớp | 11CNPM2 |
| Giảng viên | Phạm Trọng Huynh |
| Ngày thực hiện | 29/09/2026 |
| File báo cáo | [1150080095_HuynhHuuHoang_11CNPM2_BAOCAO_Word.docx](1150080095_HuynhHuuHoang_11CNPM2_BAOCAO_Word.docx) |
| Video thực hành | https://youtu.be/Y19vLefHHtg |

## Video thực hành

[![Video thực hành LAB 4 – Nmap](https://img.youtube.com/vi/Y19vLefHHtg/hqdefault.jpg)](https://youtu.be/Y19vLefHHtg)

Xem trên YouTube: [https://youtu.be/Y19vLefHHtg](https://youtu.be/Y19vLefHHtg)

## Mục tiêu bài lab

- Cài đặt và kiểm tra Nmap trên Windows 11 (kèm Npcap, Zenmap) và Kali Linux.
- Dựng mạng máy ảo Host-only an toàn, xác định đúng IP của máy quét và máy đích.
- Thực hiện host discovery, quét TCP (-sT, -sS, -sF, -sA), quét UDP, nhận diện dịch vụ (-sV), hệ điều hành (-O) và chạy NSE script kiểm tra SMB.
- Đọc và giải thích các trạng thái open / closed / filtered / unfiltered / open|filtered.
- Xuất kết quả (-oN, -oX, -oG, HTML) để làm hồ sơ bằng chứng, đánh giá rủi ro và đề xuất biện pháp phòng thủ.

## Môi trường thực hành

| Thành phần | Thực tế |
| :-- | :-- |
| Máy thật | Windows 11 Home 64-bit, RAM ≈ 8 GB; Nmap 7.991 + Npcap 1.89 |
| Ảo hóa | VMware Workstation; mạng **VMnet1 – Host-only 192.168.56.0/24** (thay cho “VirtualBox Host-Only Network” trong tài liệu) |
| Máy quét | Kali Linux 2026.2 – 192.168.56.129 – Nmap 7.99 |
| Máy đích | Metasploitable 2 – 192.168.56.130 |
| An toàn | Chỉ dùng Host-only, không Bridged; NAT của Kali chỉ bật tạm để cập nhật gói rồi gỡ; snapshot “Before-LAB4” cho cả hai VM |

```
                Windows 11 Host – 192.168.56.1
                          │
             VMnet1 – Host-only 192.168.56.0/24
         ┌────────────────┼─────────────────────┐
   Kali Linux      Metasploitable 2        VMware DHCP
 192.168.56.129     192.168.56.130        192.168.56.254
   (máy quét)         (máy đích)
```

## Tóm tắt kết quả

| Mục | Lệnh chính | Kết quả |
| :-- | :-- | :-- |
| 5. Host discovery | `sudo nmap -sn 192.168.56.0/24` | 4 host up: .1 (adapter VMnet1 của máy thật), .129 (Kali), .130 (Metasploitable 2), .254 (DHCP của VMware); phát hiện bằng ARP |
| 6.1 TCP Connect | `nmap -sT 192.168.56.130` | 23 open / 977 closed / 0 filtered – 4.74 s |
| 6.2 SYN scan | `sudo nmap -sS 192.168.56.130` | 23 open / 977 closed / 0 filtered – 4.77 s; `nmap --unprivileged -sS` bị từ chối → -sS cần quyền root |
| 6.3 FIN scan | `sudo nmap -sF --reason 192.168.56.130` | 23 open\|filtered (no-response), 977 closed (reset) – đúng hành vi RFC 793 của Linux |
| 6.4 ACK scan | `sudo nmap -sA 192.168.56.130` | 1000 unfiltered → không có firewall giữa Kali và máy đích |
| 7. UDP scan | `sudo nmap -sU --top-ports 20 --reason 192.168.56.130` | 53/udp và 137/udp open; 3 open\|filtered; 15 closed. Lần quét đầu có 5 cổng bị báo sai open\|filtered do Linux giới hạn tốc độ ICMP port unreachable |
| 8.1 Version detection | `sudo nmap -sV 192.168.56.130` | vsftpd 2.3.4, OpenSSH 4.7p1, Apache 2.2.8, Samba smbd 3.X–4.X, MySQL 5.0.51a, bindshell 1524/tcp, UnrealIRCd, Apache Tomcat (8180/tcp) |
| 8.2 OS detection | `sudo nmap -O 192.168.56.130` | Linux 2.6.X (2.6.9 – 2.6.33), cách 1 hop |
| 9.1 NSE | `--script smb-os-discovery` | Unix (Samba 3.0.20-Debian), metasploitable.localdomain |
| 9.2 NSE | `--script smb-vuln-ms17-010` | Không trả kết quả → không có dấu hiệu MS17-010 (lỗi SMBv1 của Windows); không kết luận “đã vá” |
| 10. Xuất kết quả | `-oN`, `-oX`, `-oG` + `grep`, `xsltproc` | Bộ bằng chứng dạng text, XML, grepable và trang HTML |

## Rủi ro nổi bật và khuyến nghị

| Mức | Dịch vụ | Rủi ro | Khuyến nghị |
| :-- | :-- | :-- | :-- |
| Rất cao | 1524/tcp bindshell | Shell root mở sẵn, không cần xác thực | Gỡ dịch vụ, chặn cổng, rà soát toàn vẹn hệ thống |
| Rất cao | vsftpd 2.3.4, Samba 3.0.20, UnrealIRCd | Backdoor / thực thi lệnh từ xa (CVE-2011-2523, CVE-2007-2447, CVE-2010-2075) | Nâng cấp hoặc gỡ bỏ, tắt “username map script”, tắt SMBv1 |
| Cao | Telnet 23, FTP 21, r-services 512–514 | Truyền tài khoản và dữ liệu dạng rõ, đăng nhập dựa trên tin cậy host | Tắt dịch vụ, thay bằng SSH/SFTP dùng khóa |
| Cao | MySQL 3306, PostgreSQL 5432, SMB 139/445, NFS 2049 | Dữ liệu và thư mục chia sẻ mở ra toàn mạng | Firewall default-deny, CSDL chỉ bind 127.0.0.1, giới hạn share/export |

## Nội dung file báo cáo

| Mục | Nội dung |
| :-- | :-- |
| 1 | Mục tiêu, phạm vi và quy định thực hành |
| 2 | Cài đặt Nmap trên Windows 10/11 – kèm ảnh minh chứng |
| 3 | Cài đặt Nmap và Zenmap trên Kali Linux – kèm ảnh minh chứng |
| 4 | Dựng môi trường máy ảo Host-only, lấy IP, kiểm tra kết nối – kèm ảnh minh chứng |
| 5 | Host discovery toàn dải /24 – bảng host, MAC/vendor, vai trò |
| 6 | Quét TCP: -sT, -sS, FIN, ACK – bảng so sánh và phân tích |
| 7 | Quét UDP 20 cổng phổ biến |
| 8 | Nhận diện dịch vụ (-sV), hệ điều hành (-O), Aggressive scan (-A) |
| 9 | NSE: smb-os-discovery và smb-vuln-ms17-010 |
| 10 | Xuất kết quả -oN / -oX / -oG / HTML |
| 11 | Tình huống trước và sau hardening |
| 12.2 | Trả lời 10 câu hỏi phân tích |
| 14 | Bài tập bổ sung, bản đồ dịch vụ và 4 câu hỏi phân tích bổ sung (câu 11–14) |

## Ảnh minh chứng (thư mục `file ảnh/`)

<details>
<summary>26 ảnh chụp màn hình và hình tương ứng trong báo cáo</summary>

| Tệp | Hình | Nội dung |
| :-- | :-- | :-- |
| [Screenshot 2026-09-29 135920.png](<file ảnh/Screenshot 2026-09-29 135920.png>) | 2.1 | `nmap --version` trên Windows (Nmap 7.991, Npcap 1.89) |
| [Screenshot 2026-09-29 140303.png](<file ảnh/Screenshot 2026-09-29 140303.png>) | 2.2 | `ipconfig` – VMware Network Adapter VMnet1 192.168.56.1 |
| [Screenshot 2026-09-29 140627.png](<file ảnh/Screenshot 2026-09-29 140627.png>) | 3.0 | Khởi động Kali-LAB4 |
| [Screenshot 2026-09-29 141006.png](<file ảnh/Screenshot 2026-09-29 141006.png>) | 3.1 | `sudo apt update` |
| [Screenshot 2026-09-29 141051.png](<file ảnh/Screenshot 2026-09-29 141051.png>) | 3.2 | `sudo apt install nmap` |
| [Screenshot 2026-09-29 141121.png](<file ảnh/Screenshot 2026-09-29 141121.png>) | 3.3 | `nmap --version` trên Kali (7.99) |
| [Screenshot 2026-09-29 141219.png](<file ảnh/Screenshot 2026-09-29 141219.png>) | 3.4 | Cài và mở Zenmap |
| [Screenshot 2026-09-29 140111.png](<file ảnh/Screenshot 2026-09-29 140111.png>) | 4.1a | Virtual Network Editor – VMnet1 Host-only 192.168.56.0/24 |
| [Screenshot 2026-09-29 141411.png](<file ảnh/Screenshot 2026-09-29 141411.png>) | 4.1b | Kali chỉ còn Network Adapter Host-only |
| [Screenshot 2026-09-29 141617.png](<file ảnh/Screenshot 2026-09-29 141617.png>) | 4.1c | Metasploitable 2 – Network Adapter Host-only |
| [Screenshot 2026-09-29 141915.png](<file ảnh/Screenshot 2026-09-29 141915.png>) | 4.1d | Snapshot “Before-LAB4” của Kali |
| [Screenshot 2026-09-29 141924.png](<file ảnh/Screenshot 2026-09-29 141924.png>) | 4.1e | Snapshot “Before-LAB4” của Metasploitable 2 |
| [Screenshot 2026-09-29 142614.png](<file ảnh/Screenshot 2026-09-29 142614.png>) | 4.2 | `ip -br addr` trên Kali – 192.168.56.129/24 |
| [Screenshot 2026-09-29 142629.png](<file ảnh/Screenshot 2026-09-29 142629.png>) | 4.2b | `ip route` – không có default route |
| [Screenshot 2026-09-29 142422.png](<file ảnh/Screenshot 2026-09-29 142422.png>) | 4.3 | `ifconfig` trên Metasploitable 2 – 192.168.56.130 |
| [Screenshot 2026-09-29 143051.png](<file ảnh/Screenshot 2026-09-29 143051.png>) | 4.4 | `ping -c 4 192.168.56.130` – 0% packet loss |
| [Screenshot 2026-09-29 143442.png](<file ảnh/Screenshot 2026-09-29 143442.png>) | 5.0 | Xác nhận IP máy quét trước khi quét |
| [Screenshot 2026-09-29 143528.png](<file ảnh/Screenshot 2026-09-29 143528.png>) | 5.1 | `sudo nmap -sn 192.168.56.0/24` – 4 hosts up |
| [Screenshot 2026-09-29 143717.png](<file ảnh/Screenshot 2026-09-29 143717.png>) | 5.1b | `sudo nmap -sn --reason 192.168.56.0/24` |
| [Screenshot 2026-09-29 144349.png](<file ảnh/Screenshot 2026-09-29 144349.png>) | 6.1 | `nmap -sT 192.168.56.130` |
| [Screenshot 2026-09-29 144858.png](<file ảnh/Screenshot 2026-09-29 144858.png>) | 6.2a | `nmap --unprivileged -sS` – requires root privileges |
| [Screenshot 2026-09-29 145129.png](<file ảnh/Screenshot 2026-09-29 145129.png>) | 6.2 | `sudo nmap -sS 192.168.56.130` |
| [Screenshot 2026-09-29 145800.png](<file ảnh/Screenshot 2026-09-29 145800.png>) | 6.3 | `sudo nmap -sF --reason 192.168.56.130` |
| [Screenshot 2026-09-29 145851.png](<file ảnh/Screenshot 2026-09-29 145851.png>) | 6.6 | `sudo nmap -sA 192.168.56.130` |
| [Screenshot 2026-09-29 150202.png](<file ảnh/Screenshot 2026-09-29 150202.png>) | 7.1 | `sudo nmap -sU --top-ports 20` (lần 1) |
| [Screenshot 2026-09-29 150336.png](<file ảnh/Screenshot 2026-09-29 150336.png>) | 7.1b | `sudo nmap -sU --top-ports 20 --reason` (lần 2) |

</details>

## Cấu trúc thư mục

```
LAB4/
├── README.md                                          # tệp này
├── 1150080095_HuynhHuuHoang_11CNPM2_BAOCAO_Word.docx  # báo cáo chính
├── LAB4_Nmap_HuongDan_2026.docx                       # tài liệu hướng dẫn gốc
├── LAB4_Nmap_HuongDan_2026.md                         # bản Markdown của báo cáo (trước khi chỉnh trên Word)
└── file ảnh/                                          # 26 ảnh chụp màn hình minh chứng
```

## Ghi chú

- Tài liệu hướng dẫn viết cho VirtualBox; bài làm dùng VMware Workstation, mạng VMnet1 Host-only tương đương. IP do DHCP của VMware cấp (.129, .130) nên khác sơ đồ mẫu (.10, .101).
- Mục 2–7 có ảnh chụp màn hình thật. Mục 8–10 không kèm ảnh chụp; số liệu trong bảng là kết quả chuẩn của Metasploitable 2 (đã ghi chú trong báo cáo).
- Chưa thực hiện: Xmas/NULL scan, mục 11 (so sánh trước/sau hardening), bài tập 14.1 câu 2 và câu 5.
- Phạm vi đạo đức: chỉ quét các máy ảo do chính sinh viên dựng trong mạng Host-only; không quét IP hoặc tên miền bên ngoài.
