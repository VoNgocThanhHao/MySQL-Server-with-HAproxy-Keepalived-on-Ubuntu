# Xây dựng High Available cho MySQL Server với HAproxy và Keepalived trên Ubuntu 20.04

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

    CREATE USER 'fit'@'172.20.222.2' IDENTIFIED BY '123456';

