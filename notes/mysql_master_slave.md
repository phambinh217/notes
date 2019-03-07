Dưới đây câu chuyện của anh Nguyễn Văn A trên con đường đi tìm cách tăng hiệu năng cho hệ thông tuyển dụng của công ty anh đang làm việc. Câu chuyển được chia làm 4 hồi, mời các bạn theo dõi.

## Hồi 1. Bắt đầu câu chuyện
Một cơ sở dữ liệu mysql của hệ thống tuyển dụng có 100 tables, được cài đặt trên server có CPU 2 cores, ram 2G. Những ngày đầu tiên, hệ thống tuyển dụng còn chưa phát triển, với cấu hình server này hoàn toàn thể đáp ứng được nhu cầu đọc, ghi, sao lưu dữ liệu. Tuy nhiên càng về sau, hệ thống càng lớn, các table bắt đầu đạt đến hàng triệu bản ghi, nhu cầu đọc, ghi, sao lưu diễn ra ngày càng lớn, thậm chí rất lớn, với cấu hình server hiện tại không thể đáp ứng được nhu cầu của hệ thống.
Anh Nguyễn Văn A là người quản trị cơ sở dữ liệu, được sếp giao cho nhiệm giải quyết vấn đề này. Lang thang trên Internet, anh ý tìm được 2 giải pháp:
+ Tăng cấu hình server: Tức là từ server cấu hình CPU 2 cores, ram 2G, tăng thành CPU 4 cores, ram 4G để hệ thống có thể tăng gấp đôi hiệu năng.
+ Tăng số lượng server: Tức nếu ta có 1 server 2 cores, 2G ram, thì giờ ta có 2 server với cấu hình tương tự, 1 con server để xử lý yêu cầu đọc. Con server còn lại chỉ xử lý yêu cầu ghi, truy vấn được chia đều ra 2 con server khác nhau => Hiệu năng tăng gấp đôi.

Loay hoay giữa 2 giải pháp, anh ý bắt đầu phân tích ưu nhược điểm của từng giải pháp như sau:
1. Tăng cấu hình server
Cách này đơn giản, chỉ cần liên hệ với nhà cung cấp dịch vụ server, bảo họ tăng cấu hình server lên và mình chỉ việc trả tiền cho họ. OK done, trong vòng 1 nốt nhạc hiệu năng của hệ thống đã tăng gấp đôi. Tuy nhiên với cách này, toàn bộ dữ liệu chỉ được lưu trên một máy, nếu máy này shutdown, sẽ có thể gây ra mất mát dữ liệu, ảnh hưởng tới toàn bộ hệ thống.

2. Tăng số lượng server
Cách này phức tạp, phải cấu hình nhiều thứ lằng nhằng, lại có vẻ tốn kém vì phải sắm thêm server mới. Nhưng bù lại, dữ liệu sẽ được lưu trên nhiều server, khi shutdown con này, thì còn con kia, hệ thống vẫn có thể hoạt động.

Công ty anh A đang làm thì tiền nong không thành vấn đề, lại vốn dĩ là một người thích tìm hiểu, nên anh cũng không ngại việc cấu hình lằng nhằng giữa các server. Thế nên anh ta đã quyết định sử dụng cách số 2 - Tăng số lượng server.

# Hồi 2. Tìm ra bí kíp
Khi quyết định sẽ lựa chọn cách số 2 - Tăng số lượng server. Anh A tiếp tục lang thang trên internet để tìm hiểu về cách cấu hình server để có thể áp dụng theo cách số 2 này. Thật may mắn, anh tìm thấy trong mysql có khái niệm master - slave giải quyết đúng vấn đề anh đang cần.

Theo như anh tìm hiểu, master - slave trong mysql được hiểu như là:

> Hệ thống sẽ có 2 (hoặc nhiều hơn) database giống hệt nhau, mỗi database được cài trên 1 server khác nhau. Trong số các db đó, có 1 db được gọi là master (ông chủ), các db còn lại được gọi là slave (lô lệ). Khi db master xảy ra một sự kiện làm thay đổi dữ liệu (thêm, sửa, xóa) thì db master sẽ log ra một file, các db slave thì liên tục lắng nghe file log này, và thực hiện việc thay đổi dữ liệu trên db slave như thay đổi trên db master.

Ví dụ
Trên DB master xảy ra sự kiện thêm dữ liệu mới với câu truy vấn như sau

```sql
    insert into users set email = 'phambinh217@gmail.com', password = '123456'
```

Thì ngay sau đó các db slave lắng nghe từ db master cũng sẽ thực hiện câu truy vấn tương tự.

Dựa vào cách hoạt động master - slave anh hiểu được. Anh A quyết định sử dụng db master làm db để ghi dữ liệu, còn db slave để đọc dữ liệu, đồng thời db slave cũng là db dự phòng trong trường hợp db master có vấn đề.

# Hồi 3. Cấu hình server
Anh A thực hiện demo cấu hình master - slave mysql như sau.

## Chuẩn bị
Anh tạo 2 server với 2 IP
- Server 1: 192.168.15 (đây sẽ là server master)
- Server 2: 192.168.17 (Đây sẽ là server slave)

Cả 2 server 1 và 2 đều được cài sẵn Ubuntu 16.04, mysql 5.x.
Trong Server 1, anh có tạo sẵn 1 CSLD liệu tên là `cart` (database sử dụng để demo)

## Cấu hình cho server master
Mở file `/etc/mysql/mysql.conf.d/mysqld.cnf` để cấu hình một vài thông tin cần thiết

```
sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
```

Tìm dòng `bind-address` và sửa thành

```
bind-address = 0.0.0.0
```

Tìm dòng
```
# server-id = 1
```

Bỏ dấu `#` phía trước đi


Ngay phía dưới dòng đó, ta thấy dòng

```
# log_bin = /var/log/mysql/mysql-bin.log
```

Cũng bỏ dấu `#` phía trước đi.


Khởi động lại mysql
```
sudo systemctl restart mysql
```

Tạo user để các slave có thể truy cập vào master mà đọc file log

```
mysql> CREATE USER 'slave'@'192.168.10.17' IDENTIFIED BY 'slavepass';

mysql> GRANT REPLICATION SLAVE ON *.* TO 'slave'@'192.168.10.17';

mysql>FLUSH PRIVILEGES;
```

Bằng cách chạy các lệnh trên, anh A tạo ra một user có tên là `slave`, mật khẩu là `slavepass`, user này được phép truy cập từ server có IP là `192.168.10.17`

Bạn có để ý rằng anh A tìm hiểu được các db slave sẽ đọc file log của db master không? Chính vì thế anh ý phải xem xem db master sẽ log ra file nào bằng cách chạy command sau

```
mysql> show master status;
```

Chúng ta sẽ thấy kết quả như này:
```
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000028 |      499 | scart_2019   |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```
* Lưu ý: Hãy để ý thông tin File là `mysql-bin.000028`, Position là `499`, đây là 2 thông tin quan trọng sẽ sử dụng ở phần sau

Bạn có để ý rằng anh A tìm hiểu được db master và db slave phải giống hệt nhau không? Chính vì để đảm bảo các db slave giống với db master, anh ý đã dump db master để có thể import vào các db slave sau này.

```
mysqldump -u root -p --opt cart > cart.sql
```

## Cấu hình cho server slave
Mở file `/etc/mysql/mysql.conf.d/mysqld.cnf`, cấu hình một số thông tin như sau

```
server-id = 2
log_bin = /var/log/mysql/mysql-bin.log
```

* Lưu ý: `server-id = 2` nhé

Khởi động lại mysql

```
sudo systemctl restart mysql
```

Tạo database và import database export từ db master

Tạo database

```
mysql> create database cart;
mysql> exit;
```

Import database

```
mysql -u root -p cart < /path/to/cart.sql
```

Kết nối server slave tới server master

```
mysql> STOP SLAVE;                          # Ngừng slave nếu trước đó có bật

mysql> CHANGE MASTER TO
    -> MASTER_HOST='192.168.10.15',         # Đây chính là IP của server master
    -> MASTER_USER='slave',                 # Tên tài khoản của slave tạo ở server master
    -> MASTER_PASSWORD='slavepass',         # Mật khẩu của tài khoản slave tạo ở server master
    -> MASTER_LOG_FILE='mysql-bin.000028',  # File log của server master
    -> MASTER_LOG_POS=499;                  # Position file log của server master

mysql> START SLAVE;                         # Khởi động lại slave
```
Quá trình cấu hình cho server master và server slave của anh A như vậy là kết thúc. Để đảm bảo 2 server đã kết nối với nhau. Anh A có chạy thử một câu lệnh Insert ở server master, sau đó qua server slave để truy vấn thì thấy có kết quả. Như vậy anh A đã thấy rằng 2 server đã có sự kết nối với nhau, quá trình cấu hình server master - slave thành công mỹ mãn.

# Lời kết
Bài viết trên được viết dựa trên kinh nghiệm tìm hiểu mày mò của anh Nguyễn Văn A sau 4 ngày tìm hiểu. Nếu có vấn đề gì sai xót, rất mong nhận được góp ý đến từ các bạn. Sau khi nhận được góp ý, mình sẽ báo với anh A ngay.

Cảm ơn tất cả các bạn.

---
Ở hồi sau anh A có kết hợp với dự án laravel, mời bạn đọc tiếp [Sử dụng master - slave mysql trong dự án Laravel](https://kipalog.com/posts/Su-dung-master---slave-mysql-trong-du-an-Laravel)