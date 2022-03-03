# Xây dựng High Available cho MySQL Server với HAproxy và Keepalived trên Ubuntu 20.04

## Mô hình lab:

![](https://i.imgur.com/wRA37Bt.png)

## Yêu cầu:

MySQL 1
- Hostname: mysql1
- OS: Ubuntu server 20.04
- Service : Keepalived + HAproxy + mysql-server
- Private IP: 172.20.222.1

MySQL 2
- Hostname: mysql2
- OS: Ubuntu server 20.04
- Service : Keepalived + HAproxy + mysql-server
- Private IP: 172.20.222.2

## Chuẩn bị cho các server:

    sudo apt-get install keepalived haproxy mysql-server
>
    sudo apt-get update

## Cấu hình MySQL Master - Master

![](https://i.imgur.com/6SZNhjQ.png)

### Cấu hình Master 1 - Slave 1

**Trên cả 2 server** ta tiến hành chỉnh sửa file `mysqld.cnf`:

    sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

Ghi chú hoặc xóa dòng `bind-address = 127.0.0.1`:

    #bind-address = 127.0.0.1

Tiếp theo tìm đến dòng `port = 3306`, bỏ dấu `#` ở trước và thay đổi cổng `3306` thành như sau:
- Ở máy mysql1: port = 3307
- Ở máy mysql2: port = 3308

### **Trên MySQL1 - Master 1 ( sẽ là Slave 2):**

Mở file `mysqld.cnf` và điền vào như sau :

    sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

Bỏ ghi chú các dòng:

    server-id = 1
    log_bin = /var/log/mysql/mysql-bin.log

Thoát và lưu lại thay đổi, sau đó khởi động lại dịch vụ mysql:

    service mysql restart

Tiếp theo đăng nhập vào mysql bằng tài khoản root để tạo người dùng:

    mysql -u root -p

Tạo người dùng:

    CREATE USER 'fit'@'172.20.222.2' IDENTIFIED WITH mysql_native_password BY '123456';

Cấp quyền cho người dùng:

    GRANT REPLICATION SLAVE ON *.* TO 'fit'@'172.20.222.2';

Kiểm tra thông tin của máy master:

    SHOW MASTER STATUS;

Ghi nhớ các thông tin trong khoảng khoanh đỏ:

![](https://i.imgur.com/elVi3oY.png)

### **Trên MySQL 2 -Slave 1 ( sẽ là Master 2) :**

Mở file `mysqld.cnf` và điền vào như sau :

    sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

Bỏ ghi chú và chỉnh sửa các dòng:

    server-id = 2
    log_bin = /var/log/mysql/mysql-bin.log

Thoát và lưu lại thay đổi, sau đó khởi động lại dịch vụ mysql:

    service mysql restart

Tiếp theo đăng nhập vào mysql bằng tài khoản root:

    mysql -u root -p

Chuẩn bị xong thì vào khai báo Slave 1 để nó có thể replicate data từ Master 1

    STOP SLAVE;
>
    CHANGE MASTER TO MASTER_HOST='172.20.222.1', MASTER_USER='fit', MASTER_PASSWORD='123456', MASTER_PORT=3307, MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=1528;
>
    START SLAVE;  

Sau đó kiểm tra lại trạng thái của nó:

    SHOW SLAVE STATUS\G

Nếu kết quả như bên dưới thì đã thành công:

    *************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 172.20.222.1
                  Master_User: fit
                  Master_Port: 3307
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 1528
               Relay_Log_File: fit-relay-bin.000002
                Relay_Log_Pos: 326
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
    Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 1528
              Relay_Log_Space: 534
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
    Replicate_Ignore_Server_Ids:
             Master_Server_Id: 1
                  Master_UUID: a4ebb7e6-9a9b-11ec-9818-005056a52fc6
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:

## Cấu hình Master 2 - Slave 2

### **Trên máy MySQL2 - Slave 1 & Master 2:**

Nhìn có vẻ lằng nhằng khó hiểu, nhưng thực tế là làm y chang ở trên, chỉ đổi vai trò của 2 thằng MySQL1 và MySQL2 với nhau thôi. Tạo Username và gán quyền Replication.

Tạo người dùng:

    CREATE USER 'fit'@'172.20.222.1' IDENTIFIED WITH mysql_native_password BY '123456';

Cấp quyền cho người dùng:

    GRANT REPLICATION SLAVE ON *.* TO 'fit'@'172.20.222.1';

Kiểm tra thông tin của máy master:

    SHOW MASTER STATUS;

Ghi nhớ các thông tin trong khoảng khoanh đỏ:

![](https://i.imgur.com/rOtyBHr.png)

### **Trên máy MySQL1 - Master 1 & Slave 2:**

Thực hiện khai báo cho SLAVE:

    STOP SLAVE;
>
    CHANGE MASTER TO MASTER_HOST='172.20.222.2', MASTER_USER='fit', MASTER_PASSWORD='123456', MASTER_PORT=3308, MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=680;
>
    START SLAVE;  

Sau đó kiểm tra lại trạng thái của nó:

    SHOW SLAVE STATUS\G

Nếu kết quả như bên dưới thì đã thành công:

    *************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 172.20.222.2
                  Master_User: fit
                  Master_Port: 3308
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 680
               Relay_Log_File: fit-relay-bin.000002
                Relay_Log_Pos: 326
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB:
          Replicate_Ignore_DB:
           Replicate_Do_Table:
       Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
    Replicate_Wild_Ignore_Table:
                   Last_Errno: 0
                   Last_Error:
                 Skip_Counter: 0
          Exec_Master_Log_Pos: 680
              Relay_Log_Space: 534
              Until_Condition: None
               Until_Log_File:
                Until_Log_Pos: 0
           Master_SSL_Allowed: No
           Master_SSL_CA_File:
           Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
               Master_SSL_Key:
        Seconds_Behind_Master: 0
    Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
               Last_SQL_Errno: 0
               Last_SQL_Error:
    Replicate_Ignore_Server_Ids:
             Master_Server_Id: 2
                  Master_UUID: f1627822-9a9b-11ec-9744-005056a5359a
             Master_Info_File: mysql.slave_master_info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Replica has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
     Last_SQL_Error_Timestamp:
               Master_SSL_Crl:
           Master_SSL_Crlpath:
           Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
         Replicate_Rewrite_DB:
                 Channel_Name:
           Master_TLS_Version:
       Master_public_key_path:
        Get_master_public_key: 0
            Network_Namespace:

## Kiểm tra Master - Master

### Trên máy MySQL1:

- Tạo 1 database trên máy `mysql1`:
>
    CREATE DATABASE test;

- Kiểm tra database đã được tạo chưa:
>
    SHOW DATABASES;

### Trên máy MySQL2:

- Kiểm tra database đã được tạo chưa:
>
    SHOW DATABASES;

- Tiếp theo ở máy `mysql2` thực hiện xóa database:
>
    DROP DATABASE test;

- Kiểm tra xem database đã được xóa chưa **trên cả 2 máy**:
>
    SHOW DATABASES;

![](https://i.imgur.com/DRsnuxO.png)

## Xây dựng Loadbalancer cho các server MySQL bằng HAproxy

### Cài đặt `nginx` cho cả 2 máy:

    sudo apt-get install nginx

Chỉnh sửa file bên dưới để dễ nhận biết đâu là máy `mysql1` đâu là `mysql2` *(bạn có thể bỏ qua bước này)*:

    nano /var/www/html/index.nginx-debian.html

- Ở máy `mysql1`, chỉnh sửa thành nội dung như bên dưới:

        <!DOCTYPE html>
        <html>
        <head>
        <title>Welcome to nginx!</title>
        <style>
            body {
                width: 35em;
                margin: 0 auto;
                font-family: Tahoma, Verdana, Arial, sans-serif;
            }
        </style>
        </head>
        <body>
        <h1>Welcome to MySQL1</h1>

        <p><em>Thank you for using</em></p>
        </body>
        </html>

- Ở máy `mysql2`:

        <!DOCTYPE html>
        <html>
        <head>
        <title>Welcome to nginx!</title>
        <style>
            body {
                width: 35em;
                margin: 0 auto;
                font-family: Tahoma, Verdana, Arial, sans-serif;
            }
        </style>
        </head>
        <body>
        <h1>Welcome to MySQL2</h1>

        <p><em>Thank you for using</em></p>
        </body>
        </html>

- Vào `http://ip_máy` để kiểm tra kết quả:

![](https://i.imgur.com/urgpXFb.png)

![](https://i.imgur.com/mGPTQPD.png)

### Cấu hình Keepalived

Thực hiện lần lượt cho cả 2 máy chủ:

- Thêm vào file `/etc/sysctl.conf` dòng bên dưới:

    net.ipv4.ip_nonlocal_bind=1
>
    sudo sysctl -p

- Cấu hình file `/etc/keepalived/keepalived.conf` của **keepalived**. Nếu không có sẵn thì tạo file mới rồi làm như bình thường:

    sudo nano /etc/keepalived/keepalived.conf

- Thêm vào nội dung sau:
 -   - Ở máy Mysql1:
>
    global_defs {
    router_id mysql1                            #khai báo route_id của keepalived
    }
    vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
    }
    vrrp_instance VI_1 {
    virtual_router_id 51
    advert_int 1
    priority 100
    state MASTER
    interface ens160                            #thông tin tên interface của server, bạn dùng lệnh `if>
    virtual_ipaddress {
        172.20.222.222 dev ens160           #Khai báo Virtual IP cho interface tương ứng
    }
    authentication {
        auth_type PASS
        auth_pass 123456                    #Password này phải khai báo giống nhau giữa các server keep>
        }
    track_script {
        chk_haproxy
    }
    }
    vrrp_script chk_haproxy {
    script "killall -0 haproxy"           #check pid của dịch vụ haproxy có tồn tại hay không
    interval 2                                     #thời gian lặp lại đoạn script đơn vị là second
    weight 2                                      #trọng số khấu trừ priority 2
    }
    track_script {
    chk_haproxy                             #khai báo tên đoạn script
    }

>
- 
    - Ở máy Mysql 2:
>

    global_defs {
    router_id mysql2
    }
    vrrp_script chk_haproxy {
    script "killall -0 haproxy"
    interval 2
    weight 2
    }
    vrrp_instance VI_1 {
    virtual_router_id 51
    advert_int 1
    priority 99
    state BACKUP
    interface ens160
    virtual_ipaddress {
        172.20.222.222 dev ens160
    }
    authentication {
            auth_type PASS
            auth_pass 123456
            }
    track_script {
        chk_haproxy
        }
    }



    vrrp_script chk_haproxy {
    script "killall -0 haproxy"           #check pid của dịch vụ haproxy có tồn tại hay không
    interval 2                                     #thời gian lặp lại đoạn script đơn vị là second
    weight 2                                      #trọng số khấu trừ priority 2
    }
    track_script {
    chk_haproxy                             #khai báo tên đoạn script
    }
>

Sau đó kiểm tra ip 2 máy bằng lệnh `ip a`:

![](https://i.imgur.com/b9nLwzK.png)

### Cấu hình HAproxy

Thực hiện ở cả 2 server:

Chỉnh sửa file `/etc/haproxy/haproxy.cfg`:

    sudo nano /etc/haproxy/haproxy.cfg

Thêm nội dung sau:

    frontend http-in
            bind *:8080
            default_backend app
    backend static
            balance roundrobin
            server static 172.20.222.222:80
    backend app
            balance roundrobin
            server mysql1 172.20.222.1:80 check
            server mysql2 172.20.222.2:80 check

Khởi động lại `haproxy`:

    service haproxy restart

### Kiểm tra kết quả:

Gõ ip ảo vừa tạo phía trên (`172.20.222.222`):

![](https://i.imgur.com/xBFwE8o.png)

`F5` lại để thấy sự thay đổi:

![](https://i.imgur.com/K5ECGoh.png)

## Xây dựng Loadbalancer cho các server MySQL bằng HAproxy:

Chỉnh sửa file cấu hình Haproxy **trên cả 2 server**:

    sudo nano /etc/haproxy/haproxy.cfg

Thêm các dòng sau vào:

    listen mysql-cluster
        bind 172.20.222.222:3306    #Cái này là Virtual IP , lát sẽ khai báo cùng Keepalived nhé.
        mode tcp
        balance roundrobin
            server mysql1 172.20.222.1:3307 check
            server mysql2 172.20.222.2:3308 check

Thoát là lưu lại thay đổi, sau đó khởi động lại dịch vụ Haproxy:

    sudo service haproxy restart
