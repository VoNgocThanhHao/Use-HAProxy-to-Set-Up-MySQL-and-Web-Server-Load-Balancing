# Use HAProxy to Set Up MySQL and Web-Server Load Balancing

## Mô hình

![](https://i.imgur.com/Ov0hXZ8.png)

## Yêu cầu:

Máy HA 1 (HA cho web-server): 172.20.111.0

Máy web-server 1: 172.20.111.1

Máy web-server 2: 172.20.111.2

Máy HA 2 (HA cho mysql-server): 172.20.111.99

Máy mysql-server 1: 172.20.111.3

Máy mysql-server 2: 172.20.111.4

## HAProxy với Web-Server

Tham khảo bài viết: https://github.com/VoNgocThanhHao/HAproxy-with-Web-Server

## HAProxy với MySQL

Thực hiện lần lượt trên cả 2 máy lưu trữ dữ liệu.

Cài đặt `mysql-server`:

    apt install mysql-server -y

Cấu hình lại `mysql`:

    sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

Thêm `#` vào đầu dòng để dòng thành ghi chú:

    #bind-address            = 127.0.0.1

Và bỏ ghi chú 2 dòng:

    server-id               = 1
    log_bin                 = /var/log/mysql/mysql-bin.log

***Lưu ý: với máy sql1 ta sẽ để server-id =1, còn mysql2 ta để là server-id =2.***

Thoát và lưu lại thay đổi, sau đó khởi động lại dịch vụ:

    service mysql restart

Truy cập vào `mysql` với quyền root:

    mysql -u root -p

Tạo ra 1 người dùng để giao tiếp với máy HA:

    Insert into mysql.user (Host,User,ssl_cipher,x509_issuer,x509_subject) values ('172.20.111.99','fit','abc','abc','abc');
>
    FLUSH PRIVILEGES;

Đặt mật khẩu cho người dùng:

    ALTER USER 'fit'@'172.20.111.99' IDENTIFIED WITH mysql_native_password BY 'Knn2022@';

Sau đó gán quyền cho người dùng:

    GRANT ALL PRIVILEGES ON *.* TO 'fit'@'172.20.111.99';
>
    FLUSH PRIVILEGES;

### Cấu hình cho HAProxy

**Trên máy HAproxy 2**

Cài đặt HAProxy:

    apt-get install haproxy -y

Khởi tạo HAProxy:

    sed -i "s/ENABLED=0/ENABLED=1/" /etc/default/haproxy

Kiểm tra trạng thái của dịch vụ:

    service haproxy status

![](https://i.imgur.com/0XltgJo.png)

Thêm các hostname của các máy sqlserver vào file `hosts`:

    sudo nano /etc/hosts

Thêm 2 hàng sau:

    172.20.111.3 mysql-1
    172.20.111.4 mysql-2

![](https://i.imgur.com/QaHqxb4.png)

Thoát và lưu lại thay đổi.

Tiếp theo là đổi tên file config của HAProxy để tạo backup:

    mv /etc/haproxy/haproxy.cfg{,.original}

Tạo 1 file config mới:

    nano /etc/haproxy/haproxy.cfg

Thêm vào nội dung sau:

    global
        log 127.0.0.1 local0 notice
        user haproxy
        group haproxy

    defaults
        log global
        retries 2
        timeout connect 3000
        timeout server 5000
        timeout client 5000

    listen mysql-cluster
        bind 172.20.111.99:3306
        mode tcp
        option tcpka
        balance roundrobin
        server mysql-1 172.20.111.3:3306 check
        server mysql-2 172.20.111.4:3306 check

Thoát và lưu lại. Khởi động lại dịch vụ:

    service haproxy restart

## Cấu hình MySQL Master - Master

![](https://i.imgur.com/6SZNhjQ.png)

### **Trên MySQL1 - Master 1 ( sẽ là Slave 2):**

Tiếp theo đăng nhập vào mysql bằng tài khoản root để tạo người dùng:

    mysql -u root -p

Tạo người dùng:

    CREATE USER 'staff'@'172.20.111.4' IDENTIFIED WITH mysql_native_password BY '123456';

Cấp quyền cho người dùng:

    GRANT REPLICATION SLAVE ON *.* TO 'staff'@'172.20.111.4';

Kiểm tra thông tin của máy master:

    SHOW MASTER STATUS;

Ghi nhớ các thông tin trong khoảng khoanh đỏ:

![](https://i.imgur.com/elVi3oY.png)

### **Trên MySQL 2 -Slave 1 ( sẽ là Master 2) :**

Tiếp theo đăng nhập vào mysql bằng tài khoản root:

    mysql -u root -p

Chuẩn bị xong thì vào khai báo Slave 1 để nó có thể replicate data từ Master 1

    STOP SLAVE;
>
    CHANGE MASTER TO MASTER_HOST='172.20.111.3', MASTER_USER='staff', MASTER_PASSWORD='123456', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=1528;
>
    START SLAVE;  

Sau đó kiểm tra lại trạng thái của nó:

    SHOW SLAVE STATUS\G

Nếu kết quả như bên dưới thì đã thành công:

    *************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 172.20.111.3
                  Master_User: staff
                  Master_Port: 3306
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
          Exec_Master_Log_Pos: 2071
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
                  Master_UUID: 580917f0-9b60-11ec-9e13-005056a5e8b5
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

    CREATE USER 'staff'@'172.20.111.3' IDENTIFIED WITH mysql_native_password BY '123456';

Cấp quyền cho người dùng:

    GRANT REPLICATION SLAVE ON *.* TO 'staff'@'172.20.111.3';

Kiểm tra thông tin của máy master:

    SHOW MASTER STATUS;

Ghi nhớ các thông tin trong khoảng khoanh đỏ:

![](https://i.imgur.com/rOtyBHr.png)

### **Trên máy MySQL1 - Master 1 & Slave 2:**

Thực hiện khai báo cho SLAVE:

    STOP SLAVE;
>
    CHANGE MASTER TO MASTER_HOST='172.20.111.3', MASTER_USER='staff', MASTER_PASSWORD='123456', MASTER_PORT=3306, MASTER_LOG_FILE='mysql-bin.000001', MASTER_LOG_POS=680;
>
    START SLAVE;  

Sau đó kiểm tra lại trạng thái của nó:

    SHOW SLAVE STATUS\G

Nếu kết quả như bên dưới thì đã thành công:

    *************************** 1. row ***************************
               Slave_IO_State: Waiting for source to send event
                  Master_Host: 172.20.111.4
                  Master_User: staff
                  Master_Port: 3306
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
          Exec_Master_Log_Pos: 2071
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
                  Master_UUID: 59a0f193-9b60-11ec-9c61-005056a5627f
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


## ***Các bước đã hoàn thành, bây giờ chi việc vào ip máy HA 1 để kiểm tra thôi!!!!***
