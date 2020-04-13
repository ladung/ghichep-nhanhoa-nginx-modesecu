# Hướng dẫn cấu hình thống kê Jitsi

## Bản mới

Thay đổi thông số sau trong file cấu hình `/etc/jitsi/videobridge/config`

`JVB_OPTS="--apis=rest"`

Rồi restart lại service

`service jitsi-videobridge2 restart`

Kiểm tra ở địa chỉ sau:

`http://ip-jitsi:8080/colibri/stats`

## Bản cũ

Thêm cấu hình sau nếu chưa có vào file `/etc/jitsi/videobridge/sip-communicator.properties`

`org.jitsi.videobridge.ENABLE_STATISTICS=true`

Chỉnh sửa file cấu hình `/etc/jitsi/videobridge/config`

`JVB_OPTS="--apis=rest,xmpp"`

Sau đó restart service và kiểm tra như trên.

`service jitsi-videobridge restart`