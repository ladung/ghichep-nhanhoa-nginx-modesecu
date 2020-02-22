# Hướng dẫn xử lý vlan cho các máy ảo trên SolusVM KVM

## Mục tiêu

Đối với các máy chủ KVM được quản lý bởi solusvm, mặc định sẽ chỉ được khai báo và sử dụng một dải địa chỉ trùng với dải địa chỉ khai báo trong bridge mà ta setup.

Để sử dụng nhiều dải địa chỉ, ta sẽ phải khai báo thêm các vlan.

## Các bước thực hiện

Với các máy chủ KVM mặc định được cài từ solusvm, thường người dùng sẽ tạo ra `br0` với ip là ip chính của máy ảo. Điều này có nghĩa rằng các máy ảo trên đó sẽ sử dụng dải ip của br0 và port phía sw được sử dụng ở mode access.

<img src="https://i.imgur.com/cf0sKJ7.png">

Ở đây `kvm101.0` chính là port ở đầu br của máy ảo.

Trước tiên, ta sẽ chỉnh cấu hình trong file config network của máy chủ kvm

Chỉnh config của card mạng vật lý đang được gán với br0

```
[root@solus network-scripts]# cat ifcfg-eth0
NAME="eth0"
DEVICE=eth0
ONBOOT="yes"
BOOTPROTO="none"
TYPE="Ethernet"
```

Ta sẽ tạo ra các sub interface tương ứng với các vlan có trong hệ thống. Ở đây mình có 2 vlan là 11 và 12, trong đó `vlan11` có dải ip mà br0 đang sử dụng. Ta sẽ tạo ra `eth0.11` và `eth0.12`

```
[root@solus network-scripts]# cat ifcfg-eth0.11
VLAN=yes
BRIDGE=vlan11
DEVICE=eth0.11
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
```

```
[root@solus network-scripts]# cat ifcfg-eth0.12
VLAN=yes
BRIDGE=vlan12
DEVICE=eth0.12
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
```

Ta có thể thấy các sub interface này được gán với bridge vlan11 và vlan12. Tiếp tục ta sẽ tạo 2 bridge này.

```
[root@solus network-scripts]# cat ifcfg-vlan11
DEVICE=vlan11
TYPE=Bridge
BOOTPROTO=none
ONBOOT=yes
IPADDR=10.10.11.172
PREFIX=24
GATEWAY=10.10.11.1
DNS1=8.8.8.8
```

```
[root@solus network-scripts]# cat ifcfg-vlan12
DEVICE=vlan12
TYPE=Bridge
BOOTPROTO=none
ONBOOT=yes
IPADDR=10.10.12.172
PREFIX=24
```

Lưu ý : Tất cả các bridge này đều phải có IP bởi dhcp được chạy trên chính node vật lý này.

Cuối cùng đừng quên xóa cấu hình có sẵn của `br0`

`rm /etc/sysconfig/network-scripts/ifcfg-br0`

Tiếp đến, ta sẽ tiến hành tắt toàn bộ máy ảo để cập nhật lại config. Sau khi máy ảo được tắt, ta sẽ update config trong tab `Dashboard\Virtual Servers\server-name\Custom Config`

Tích vào `Enable Custom Config`. Sau đó copy paste custom config (lưu ý copy toàn bộ config). Sửa block interfaces

```
<interface type='bridge'>
 <model type='virtio' />
 <source bridge='br0'/>
 <target dev='kvm101.0'/>
 <mac address='00:16:3c:5a:b5:b2'/>
</interface>
```

Thay `<source bridge='br0'/>` thành `<source bridge='vlan11'/>`

Sau khi thay xong, tiến hành save lại.

Tiếp tục, ta sẽ chỉnh config trên sw từ mode access về trunk, allow các vlan mong muốn và restart lại network của node kvm rồi kiểm tra lại.

Nếu các bridge vlan đã up hết, ta sẽ bật lại các máy ảo sau đó kiểm tra.

