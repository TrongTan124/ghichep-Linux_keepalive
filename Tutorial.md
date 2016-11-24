# 1. Giới thiệu
- Keepalive được sử dụng cho IP Failover (còn gọi là IP float) giữa 2 server. Nó làm việc dựa vào VRRP (Virtual Router Redundancy Protocol)
- Mục đích cho việc sử dụng keepalive là việc bạn gán IP float cho 02 server để trong trường hợp có 01 server nhận ip float bị sự cố thì sẽ tự động gán ip float cho server còn lại.

# 2. Cài đặt

Tôi sử dụng distro linux là Ubuntu 14.04 64bit trong tất cả các bài test của tôi.

# 3. Cấu hình

Để cấu hình với tính năng đơn gian là gán IP VIP thì rất dễ dàng, tạo file config với cấu hình như sau:

- Trên node Master (sẽ đặt priority cao hơn để ưu tiên gán IP VIP cho node này)
```sh
# cat /etc/keepalived/keepalived.conf 
vrrp_instance VI_1 {
    state MASTER			# have 3 states: MASTER, BACKUP, FAULT
    interface eth0			# interface add IP VIP
    virtual_router_id 51	# ID virtual router
    priority 100			# priority to set check Master-backup
    advert_int 1
    authentication {
        auth_type PASS		# method authen, this case, use password
        auth_pass tan124	# password authen
    }
    virtual_ipaddress {
        172.16.69.94		# IP VIP
    }
}
```

- Trên node Backup
```sh
# cat /etc/keepalived/keepalived.conf 
vrrp_instance VI_1 {
    state BACKUP			# have 3 states: MASTER, BACKUP, FAULT
    interface eth0			# interface add IP VIP
    virtual_router_id 51	# ID virtual router
    priority 99				# priority to set check Master-backup
    advert_int 1			# time check 
    authentication {
        auth_type PASS		# method authen, this case, use password
        auth_pass tan124	# password authen
    }
    virtual_ipaddress {
        172.16.69.94		# IP VIP
    }
}
```

Thực hiện cấu hình nâng cao để phục vụ các tính năng theo nhu cầu thì phải hiểu về cơ chế làm việc của keepalived

- Đầu tiên là script check status, trả về 0 nếu mọi thứ OK, trả về 1 hoặc lớn hơn 0 nếu có gì đó NOK
```sh
vrrp_script script_name {
  script       "program_path arg ..."
  interval i  # Run script every i seconds
  fall f      # If script returns non-zero f times in succession, enter FAULT state
  rise r      # If script returns zero r times in succession, exit FAULT state
  timeout t   # Wait up to t seconds for script before assuming non-zero exit code
  weight w    # Reduce priority by w on fall - giá trị này khá quan trọng khi thiết lập độ ưu tiên priority, nó sẽ giảm một giá trị w để làm sao thấp hơn node backup. hoặc backup tăng 1 giá trị w
}
```

- Tiếp đến ta thực hiện action khi check status. Action sẽ gọi tới với chỉ dẫn notify (toàn bộ các status trả về). hoặc chỉ định notify với từng trạng thái
```sh
vrrp_instance MyVRRPInstance {
  state MASTER
  interface eth0
  virtual_router_id 5
  priority 200
  advert_int 1
  virtual_ipaddress {
    192.168.1.1/32 dev eth0
  }
  track_script {
    chk_myscript
  }
  notify /usr/local/bin/keepalivednotify.sh
}

Keepalived passes the following 3 parameters to the notify script
The script is called after any state change with the following parameters:

$1 = “GROUP” or “INSTANCE”
$2 = name of group or instance
$3 = target state of transition (“MASTER”, “BACKUP”, “FAULT”)
Here is a sample script keepalivednotify.sh:

#!/bin/bash

TYPE=$1
NAME=$2
STATE=$3

case $STATE in
        "MASTER") /etc/init.d/apache2 start
                  exit 0
                  ;;
        "BACKUP") /etc/init.d/apache2 stop
                  exit 0
                  ;;
        "FAULT")  /etc/init.d/apache2 stop
                  exit 0
                  ;;
        *)        echo "unknown state"
                  exit 1
                  ;;
esac
```

- Ngoài ra có thể thực hiện rất nhiều action với việc notify gọi script. 
Ví dụ có thể tham khảo tại [đây](http://stackoverflow.com/questions/28931017/keepalived-script-makes-failover-go-insane) 


- Có thể sử dụng thêm track_interface để kiểm tra việc up, down của interface
```sh
vrrp_instance test_instance {
   interface eth0

   track_interface {
     eth0
     eth1
   }
...
}
```

- Thêm một variable quan trọng nữa là nopreempt, nó sẽ chuyển trạng thái của keepalived từ FAULT -> BACKUP nếu script check status trả về giá trị 0

- Có cấu hình global_defs sử dụng để gửi email
```sh
global_defs {

   notification_email {
       admin@example.com
   }
   notification_email_from noreply@example.com
   smtp_server 127.0.0.1
   smtp_connect_timeout 60
   router_id LVS_DEVEL
}
```

# Loadbalancer

Có thể cấu hình keepalived làm loadbalacer với khai báo sau
```sh
virtual_server 10.0.0.1 80 {
    delay_loop 6	# time health check
    lb_algo rr		# algorithm: round-robin (rr), weighted round-robin (wrr), least-connection (lc), weighted least-connection (wlc), ...
    lb_kind NAT		# determines routing method: NAT, DR
    protocol TCP
	persistence_timeout 9600

    real_server 192.168.1.20 80 {
        TCP_CHECK {					# check availability
                connect_timeout 10
        }
    }
    real_server 192.168.1.21 80 {
		weight 1
        TCP_CHECK {
                connect_timeout 10
				connect_port    80
        }
    }
    real_server 192.168.1.22 80 {
		weight 1
        TCP_CHECK {
                connect_timeout 10
				connect_port    80
        }
    }
    real_server 192.168.1.23 80 {
		weight 1
        TCP_CHECK {
                connect_timeout 10
				connect_port    80
        }
    }
}
```

# keepalived config for ELK stack 2 node

- Config keepalived
```sh
vrrp_script chk_service {
    script /root/scripts/check-service.sh
    interval 2
#	weight 2
	fall 2
	rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 99
    advert_int 1
nopreempt
    authentication {
        auth_type PASS
        auth_pass tan124
    }
    virtual_ipaddress {
        172.16.69.94
    }
  track_script {
    chk_service
  }
notify /root/scripts/keepalived.state.sh
}
```

- script check
```sh
# cat check-service.sh 
#!/bin/bash

function get_ip {
	ip_eth0=`/sbin/ifconfig eth0 | grep 'inet addr:' | cut -d: -f2 | awk '{ print $1}'`
        echo $ip_eth0
}

# check telnet port, if success return 0, if false return 1
function check_telnet {
	local ip port
	ip=$1
	port=$2
	result=`nc -z -w2 $ip $port`
	if [ $? == 0 ]; then
		echo 0
	else
		echo 1
	fi
}

port_logstash=5044
port_elasticsearch=9200
ip_eth0=`get_ip`
if [[ `check_telnet $ip_eth0 $port_logstash` == 0 && `check_telnet $ip_eth0 $port_elasticsearch` == 0 ]]; then
	exit 0
else
	exit 1
fi
```

- script notify
```sh
# cat keepalived.state.sh 
#!/bin/bash

TYPE=$1
NAME=$2
STATE=$3

echo $STATE > /var/run/keepalived.state
```

# Tham khảo
- [https://docs.oracle.com/cd/E37670_01/E41138/html/section_hxz_zdw_pr.html](https://docs.oracle.com/cd/E37670_01/E41138/html/section_hxz_zdw_pr.html)
- [https://tobrunet.ch/2013/07/keepalived-check-and-notify-scripts/](https://tobrunet.ch/2013/07/keepalived-check-and-notify-scripts/)
- [http://www.linux-admins.net/2015/02/keepalived-using-unicast-track-and.html](http://www.linux-admins.net/2015/02/keepalived-using-unicast-track-and.html)
- [https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Load_Balancer_Administration/ch-initial-setup-VSA.html](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux/7/html/Load_Balancer_Administration/ch-initial-setup-VSA.html)
