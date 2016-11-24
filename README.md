Đây là ghi chép của tôi trong quá trình tìm hiểu về keepalived
====

*Có thể những ghi chép này được cóp nhặt vụn vặt từ nhiều nguồn, tôi sẽ để lại link gốc những nguồn mà tôi tham khảo.*

# Giới thiêu

keepalived được sử dụng trong các hệ thống cần IP VIP cho việc kết nối, đáp ứng được yêu cầu high available trong trường hợp IP VIP được gán cho một server mà server đó bị lỗi, 
keepalived sẽ thực hiện chuyển IP VIP sang server dự phòng để giảm thời gian gián đoạn dịch vụ

Ngoài keepalive có thể sử dụng pacemaker, heartbeat để tạo IP VIP, nhưng keepalive sử dụng đơn giản hơn rất nhiều.

----
Mọi ý kiến đóng góp có thể phản hồi theo địa chỉ sau:
- Skype: crazyman12487
- Gmail: nguyentrongtan124@gmail.com