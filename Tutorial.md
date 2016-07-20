##1. Giới thiệu
- Keepalive được sử dụng cho IP Failover (còn gọi là IP float) giữa 2 server. Nó làm việc dựa vào VRRP (Virtual Router Redundancy Protocol)
- Mục đích cho việc sử dụng keepalive là việc bạn gán IP float cho 02 server để trong trường hợp có 01 server nhận ip float bị sự cố thì sẽ tự động gán ip float cho server còn lại.