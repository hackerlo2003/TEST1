1. Cấu hình VM1 (VPN Gateway & DNS Forwarder)VM1 đóng vai trò là "cửa ngõ" nhận truy vấn DNS từ Client, chuyển tiếp nó qua đường truyền VPN an toàn đến DNS Server (VM2).Cấu hình OpenVPN ServerTạo file cấu hình server tại /etc/openvpn/server.conf:textdev tun

Hãy thận trọng khi sử dụng mã.Định tuyến và Firewall (iptables)Để VM1 nhận diện và xử lý lưu lượng, bạn cần bật IP Forwarding và cấu hình luật tường lửa:bash# Bật IP Forwarding
sysctl -w net.ipv4.ip_forward=1

# Cho phép chuyển tiếp dữ liệu giữa ens160 và tun0
iptables -A FORWARD -i ens160 -o tun0 -j ACCEPT
iptables -A FORWARD -i tun0 -o ens160 -m state --state RELATED,ESTABLISHED -j ACCEPT
Hãy thận trọng khi sử dụng mã.Cấu hình DNS Forwarder (Sử dụng Dnsmasq)Cài đặt dnsmasq và cấu hình tại /etc/dnsmasq.conf:text# Lắng nghe truy vấn từ mạng client
interface=ens160
listen-address=172.16.27.102

# Không đọc file /etc/resolv.conf của hệ thống để tránh ra Internet
no-resolv
no-hosts

# Chuyển tiếp toàn bộ truy vấn DNS đến VM2 (DNS Server nội bộ) qua VPN

Hãy thận trọng khi sử dụng mã.Sau khi kết nối thành công, VM2 sẽ nhận IP 10.8.0.2 trên giao diện tun0.Cấu hình DNS Server (Sử dụng BIND9)Cấu hình file chính /etc/bind/named.conf.options:textoptions {
    directory "/var/cache/bind";

    # Chỉ lắng nghe truy vấn từ interface VPN
    listen-on { 10.8.0.2; };
    listen-on-v6 { none; };

    # Khóa chặt, không cho phép forward đi đâu nữa để đảm bảo cô lập
    allow-query { 10.8.0.0/24; };
    recursion no; 
};
Hãy thận trọng khi sử dụng mã.Cấu hình Zone tại /etc/bind/named.conf.local:textzone "lab.local" {
    type master;
    file "/etc/bind/db.lab.local";
};
Hãy thận trọng khi sử dụng mã.Tạo file cơ sở dữ liệu zone /etc/bind/db.lab.local:text$TTL    604800
@       IN      SOA     ns1.lab.local. root.lab.local. (
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.lab.local.
ns1     IN      A       10.8.0.2
Hãy thận trọng khi sử dụng mã.3. Cấu hình Client (Mạng 172.16.27.0/24)Môi trường này giả lập máy nạn nhân bị nhiễm mã độc sinh ra DNS Beacon.IP: Thuộc dải 172.16.27.0/24 (Ví dụ: 172.16.27.50).DNS Server: Chỉ định duy nhất 172.16.27.102 (VM1).
