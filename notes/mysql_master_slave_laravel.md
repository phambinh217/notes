Tiếp tục câu chuyện của anh Nguyễn Văn A ở bài trước

Đọc thêm [Phần 1](/notes/mysql_master_slave.md)

# Hồi 4. Sử dụng thực tế với laravel.
Demo thì xong rồi, giờ thì phải áp dụng được vào thực tế. Anh A quyết định deploy một project laravel (5.5) và connect vào database `cart` mà anh đã tạo ở hồi 3.
Lại tiếp tục lần mò internet, anh thấy rằng laravel hỗ trợ `read` và `write` tại 2 database khác nhau. Cách cấu hình thì cực kỳ đơn giản.
Chi tiết bạn có thể xem tại: https://laravel.com/docs/5.5/database#read-and-write-connections

"Thật là quá tuyệt vời đi" - anh A kêu lênh sung sướng.

Anh A bắt đầu việc cấu hình cho dự án laravel của mình.
Ý tưởng của anh rằng, database master sẽ chỉ sử dụng để `write`, trong khi database `slave` sẽ chỉ sử dụng để `read`, source code laravel của anh sẽ để trên Server 1 (nơi cùng chưa database master).

Các bước cấu hình cho dự án laravel được anh A thực hiện như sau:

Tạo một tài khoản trên server slave

```
mysql> CREATE USER 'nguyenvana'@'%' IDENTIFIED BY 'password';

mysql> GRANT ALL PRIVILEGES ON *.* TO 'nguyenvana'@'%' WITH GRANT OPTION;

mysql> FLUSH PRIVILEGES;
```

Bằng cách chạy các lệnh trên, anh A đã tạo ra một user có tên là `nguyenvana`, mật khẩu là `password`



Mở file `config/database.php`, tìm đến đoạn config kết nối với mysql, anh A sửa thành
```php
'mysql' => [
    'driver' => 'mysql',

    # Write trên database master
    # Vì database master và source code trên cùng 1 Server
    # nên anh A giữ nguyên tài khoản kết nối đến mysql ban đầu
    'write' => [
        'host' => env('DB_HOST', '127.0.0.1'),
        'port' => env('DB_PORT', '3306'),
        'username' => env('DB_USERNAME', 'forge'),
        'password' => env('DB_PASSWORD', ''),
    ],

    # Read trên database slave
    # Vì database slave và source code nằm tại 2 server khác nhau
    # nên anh A phải sử dụng tài khoản slave vừa tạo ở đây
    'read' => [
        'host' => env('DB_READ_HOST'), # 192.168.10.17
        'port' => env('DB_READ_PORT', '3306'),
        'username' => env('DB_READ_USERNAME', 'forge'), # nguyenvana
        'password' => env('DB_READ_PASSWORD', ''), # password
    ],

    # Vì tại 2 server master và slave, tên database đều là `cart`
    # Vậy nên anh A giữ nguyên thông tin cấu hình này
    # không đưa vào 2 thông tin cầu hình `read` và `write
    'database' => env('DB_DATABASE', 'forge'),

    'unix_socket' => env('DB_SOCKET', ''),
    'charset' => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'prefix' => '',
    'strict' => true,
    'engine' => null,
],
```


Kiểm tra đảm bảo laravel đã `write` và `read` trên server master và slave tương ứng.

```php
// Create là hành động write, vậy sẽ xảy ra trên db master
$productOnMaster = Product::create([
    'title' => 'Đây là sản phẩm test'
]);

// Find là hành động read, vậy sẽ xảy ra trên db slave
$productOnSlave = Product::find($productOnMaster->id);

// Nếu lấy được dữ liệu trên slave
// và ID của product trên slave == ID của product trên master
if ($productOnSlave && $productOnSlave->id == $productOnMaster->id) {
    echo "It work";
} else {
    echo "Fails";
}
```

# Lời kết
Bài viết trên được viết dựa trên kinh nghiệm tìm hiểu mày mò của anh Nguyễn Văn A sau 4 ngày tìm hiểu. Nếu có vấn đề gì sai xót, rất mong nhận được góp ý đến từ các bạn. Sau khi nhận được góp ý, mình sẽ báo với anh A ngay.

Cảm ơn tất cả các bạn.