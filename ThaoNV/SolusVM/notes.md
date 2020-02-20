# Một số ghi chép với SolusVM

## 1. DHCP

Solus sử dụng service dhcpd trên Linux để  cung cấp dhcp cho máy ảo. Mỗi khi có IP được thêm vào máy ảo hoặc máy ảo mới được tạo ra, sẽ có một cấu hình được thêm vào file `/etc/dhcpd.conf`

```
[root@solus ~]# cat /etc/dhcpd.conf
# Written by SolusVM - 2020-02-19 04:52:50

subnet 0.0.0.0 netmask 0.0.0.0 {
authoritative;
default-lease-time 21600000;
max-lease-time 432000000;
}
ddns-update-style interim;

host kvm101.0 {
hardware ethernet 00:16:3c:2a:fb:b4;
option routers 10.10.12.1;
option subnet-mask 255.255.255.0;
fixed-address 10.10.12.176;
option domain-name-servers 8.8.8.8,8.8.4.4;
}

host kvm102.0 {
hardware ethernet 00:16:3c:40:0a:5b;
option routers 10.10.12.1;
option subnet-mask 255.255.255.0;
fixed-address 10.10.12.177;
option domain-name-servers 8.8.8.8,8.8.4.4;
}
```

Việc thay đổi này được Solus cập nhật mỗi khi máy ảo tắt đi bật lại (trường hợp reboot từ trong máy ảo không có tác dụng)
Nghĩa là dù bạn có cập nhật bằng tay ở file dhcpd.conf khi máy ảo đang chạy, restart dhcp service và network của máy ảo để request dhcp mới thì khi máy ảo được tắt đi và bật lại, solus sẽ tự cập nhật IP được gán cho máy ảo trên giao diện vào file này.

Tóm lại, IP của máy ảo mà bạn nhìn thấy trên giao diện sẽ là IP mà solus dùng để fix vào file dhcp ở host KVM theo địa chỉ MAC.

## 2. IP Stealing & ARP Attack

Vì không sử dụng cơ chế Port security như OpenStack nên mặc định, máy ảo có thể gán bất cứ IP nào được định nghĩa trong block.

Solus cung cấp 1 option đó là `IP Stealing & ARP Attack`. Ta set nó trong phần Node -> Settings.

Khi tùy chọn này được enable. Các máy ảo sẽ chỉ sử dụng được IP được gán cho nó, khi ta cố tình đổi sang một IP khác trong dải, nó sẽ không thể truy cập ra ngoài cũng như bên ngoài không thể truy cập vào trong.

Việc này thực chất được Solus thực hiện thông qua `ebtables` - một công cụ dùng để filter các traffic đi qua bridge trên linux.

Đây là rule khi chưa enable tùy chọn trên

```
[root@solus ~]# ebtables -L
Bridge table: filter

Bridge chain: INPUT, entries: 0, policy: ACCEPT

Bridge chain: FORWARD, entries: 0, policy: ACCEPT

Bridge chain: OUTPUT, entries: 0, policy: ACCEPT
```

Ta thấy không có rule nào cả.

Và đây là các rule khi ta enable tùy chọn này

```
[root@solus ~]# ebtables -L
Bridge table: filter

Bridge chain: INPUT, entries: 2, policy: ACCEPT
-i kvm101.0 -j kvm101.0
-i kvm102.0 -j kvm102.0

Bridge chain: FORWARD, entries: 2, policy: ACCEPT
-i kvm101.0 -j kvm101.0
-i kvm102.0 -j kvm102.0

Bridge chain: OUTPUT, entries: 0, policy: ACCEPT

Bridge chain: kvm101.0, entries: 6, policy: DROP
-p IPv4 --ip-proto udp --ip-sport 67:68 -j ACCEPT
-p IPv4 --ip-proto udp --ip-dport 67:68 -j ACCEPT
-p ARP --arp-op Request -j ACCEPT
-p IPv4 --ip-src 10.10.12.176 -j ACCEPT
-p IPv4 --ip-dst 10.10.12.176 -j ACCEPT
-p ARP --arp-op Reply --arp-ip-src 10.10.12.176 -j ACCEPT

Bridge chain: kvm102.0, entries: 6, policy: DROP
-p IPv4 --ip-proto udp --ip-sport 67:68 -j ACCEPT
-p IPv4 --ip-proto udp --ip-dport 67:68 -j ACCEPT
-p ARP --arp-op Request -j ACCEPT
-p IPv4 --ip-src 10.10.12.177 -j ACCEPT
-p IPv4 --ip-dst 10.10.12.177 -j ACCEPT
-p ARP --arp-op Reply --arp-ip-src 10.10.12.177 -j ACCEPT
```

Ta dễ dàng nhận ra việc filter theo port của VM trên bridge. Ngoài việc mở port dhcp là 67 và 68. Nó chặn hết toàn bộ traffic không có dest hoặc src từ ip của máy ảo tương ứng.
