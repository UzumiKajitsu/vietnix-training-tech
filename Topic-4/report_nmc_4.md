# NGUYỄN MINH CHIẾN - Nội dung 09/09/2025
## Reverse Proxy
### Khái niệm
- Reverse Proxy là một hệ thống được đặt trước các nhóm Server, đóng vai trò như người trung gian để phục vụ việc giao tiếp giữa Client và Web Server. Cụ thể, Reverse Proxy sẽ chặn yêu cầu từ Client (như trình duyệt Web), sau đó giao tiếp với Web Server thay cho Client đó.

- Trong các giao tiếp Internet tiêu chuẩn, Client sẽ trực tiếp kết nối với Web Server, chúng gửi yêu cầu và chờ đợi kết quả phản hồi trực tiếp từ Web Server. Nhưng với hệ thống có Reverse Proxy, thì Reverse Proxy sẽ tiếp nhận yêu cầu từ Client, sau đó gửi yêu cầu này đến Web Server. Kết quả phản hồi từ Web Server được gửi trả về lại cho Reverse Proxy, sau đó mới được truyền đến Reverse Proxy.
### Mô hình Reverse Proxy với Nginx và Apache
- Khi áp dụng mô hình này với hai máy chủ web là Nginx và Apache, Nginx thường được đặt ở vị trí tiền tuyến, đóng vai trò là Reverse Proxy, trong khi Apache hoạt động như máy chủ backend xử lý các ứng dụng web. Nginx đứng trước Apache bởi nhiều lý do kỹ thuật và vận hành:
    - Với sử dụng kiến thức đơn luồng, kỹ thuật bất đồng bộ (asynchronous), Nginx đạt được hiệu suất và ổn định cao.
    - Nginx tiêu tốn rất ít tài nguyên dù phải phục vụ lượng lớn request đồng thời, giúp giảm tải rất nhiều cho Apache.
    - Nginx có thể thực hiện các chức năng như mã hóa HTTPS cho toàn bộ các miền và chứng chỉ SSL, load balancing, caching, và bảo mật tường lửa ứng dụng cơ bản, đảm bảo các yêu cầu từ client được kiểm soát trước khi đi đến Apache.
    - Apache phục vụ  xử lý các rewrite rule phức tạp, hỗ trợ .htaccess và tương thích với các ứng dụng như WordPress hay Laravel, được giữ làm máy chủ backend phía sau.
### Cài đặt Apache
- Để cài đặt Apache, sử dụng các lệnh sau:
```bash
sudo apt update
sudo apt install apache2
sudo systemctl enable --now apache2
```
- Sau khi cài đặt, vì mặc định Apache sử dụng port 80 tương tự như N Nginx nên khi khởi động sẽ bị lỗi do Nginx đã chiếm dụng tiến trình. Vì vậy phải đổi port của Apache sang port khác (ví dụ 8080). Truy cập file cấu hình `/etc/apache2/ports.conf` để thay đổi:
```bash
Listen 8080
```
- Sau đó reload lại Apache và kiểm tra bằng truy cập lên trình duyệt hoặc status:
```bash
sudo systemctl restart apache2
sudo systemctl status apache2
```
![alt text](./image-topic4/image-3.png)
![alt text](./image-topic4/image.png)

## Xây dựng website với hệ thống Reverse Proxy
Cấu hình `/etc/apache2/ports.conf` để APache chỉ lắng nghe các port của các website cấu hình trong vhost:
```bash
Listen 8080 # port của Wordpress
Listen 8081 # port của Laravel
Listen 8090 # port của Default vhost (dùng cho câu sau)
```
### WordPress
#### Cấu hình Apache
- Tạo Virtual Host chứa website Wordpress tại thư mục `/etc/apache2/sites-available/wordpress.conf` 
    ```bash
    <VirtualHost 127.0.0.1:8080>
        ServerName wp.chiennguyen.vietnix.tech
        DocumentRoot /var/www/wordpress
        <Directory /var/www/wordpress>
            AllowOverride All
            Require all granted
        </Directory>
    </VirtualHost>
    ```
    - `VirtualHost`: Máy chủ ảo lắng nghe dựa trên địa chỉ IP và port được cấu hình. Ở đây IP được cấu hình là `127.0.0.1` (localhost) để  Apache chỉ nhận request từ chính máy chủ, nghĩa là chỉ có Nginx hoặc các ứng dụng nội bộ mới có thể truy cập giúp tăng bảo mật và phù hợp với mô hình reverse proxy. Nginx sẽ nhận request từ client và chuyển tiếp tới Apache trên localhost.
    - `ServerName`: Khai báo tên miền hoặc IP để Apache nhận diện (website này sẽ phục vụ khi người dùng truy cập domain/IP này).
    - `DocumentRoot`: Thư mục gốc chứa mã nguồn website WordPress.
    - `<Directory>` Quy định quyền và cách xử lý cho thư mục:
        - `AllowOverride All`: Cho phép sử dụng file `.htaccess` để ghi đè cấu hình.
        - `Require all granted`: Cho phép truy cập vào tất cả thư mục.
- Sau khi cấu hình, kích hoạt vhost và kiểm tra syntax, sau đó khởi động lại apache2
    ```bash
        sudo a2ensite wordpress.conf
        sudo apache2ctl configtest
        sudo systemctl reload apache2
    ```
- Cấu hình `.htaccess` trong `/var/www/wordpresss/`. `.htaccess` dùng để cấu hình Apache tại thư mục web mà không cần sửa file chính, chủ yếu để URL thân thiện và hỗ trợ HTTPS qua reverse proxy.
    ```bash
    # BEGIN WordPress
    <IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteRule .* - [E=HTTP_AUTHORIZATION:%{HTTP:Authorization}]
    RewriteBase /
    RewriteRule ^index\.php$ - [L]
    RewriteCond %{REQUEST_FILENAME} !-f
    RewriteCond %{REQUEST_FILENAME} !-d
    RewriteRule . /index.php [L]
    </IfModule>
    SetEnvIf X-Forwarded-Proto "https" HTTPS=on
    # END WordPress
    ```
- `<IfModule>:` Chỉ chạy các rule rewrite nếu module mod_rewrite bật.
- `RewriteEngine On`: Bật rewrite.
- `RewriteRule .*`: Giữ header Authorization.
- `RewriteBase /`:Thư mục gốc cho rewrite.
- `RewriteRule ^index\.php$ - [L]`: Không rewrite index.php.
- `RewriteCond %{REQUEST_FILENAME} !-f và !-d`: Điều kiện file/thư mục không tồn tại.
- `RewriteRule . /index.php [L]`:Gửi tất cả request về index.php.
- `SetEnvIf X-Forwarded-Proto "https" HTTPS=on`: Khi dùng Nginx reverse proxy, báo cho Apache biết site đang dùng HTTPS.

#### Cấu hình Nginx
- Cấu hình Nginx làm reverse proxy cho website WordPress chạy trên Apache backend. cấu hình 2 block server qua file `/etc/nginx/sites-available/wp.chiennguyen.vietnix.tech` riêng biệt cho truy cập HTTP và HTTPS theo yêu cầu đề bài (Không cấu hình redirect từ HTTP -> HTTPS).
    ```bash
        server {
            listen 80;
            server_name wp.chiennguyen.vietnix.tech;

            location / {
                proxy_pass http://127.0.0.1:8080;  # Apache backend chạy HTTP
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Proto $scheme;
            }
        }
    ```
    - `listen 80`: lắng nghe HTTP.
    - `server_name`: Domain của website.
    - `proxy_pass`: gửi tất cả request tới Apache Backend tại 127.0.0.1:8080.
    - `proxy_set_header Host $host`: giữ nguyên header Host, để - WordPress biết domain đúng.
    - `X-Real-IP & X-Forwarded-For`: gửi IP client gốc tới Apache (không bị mất IP khi qua proxy).
    - `X-Forwarded-Proto`: gửi scheme (http hoặc https) tới Apache, để WordPress nhận biết HTTPS đúng khi cần.

    ```bash
    server {
        listen 443 ssl;
        server_name wp.chiennguyen.vietnix.tech;

        ssl_certificate     /etc/nginx/ssl/wp.crt;
        ssl_certificate_key /etc/nginx/ssl/wp.key;
    
        add_header Content-Security-Policy "upgrade-insecure-requests" always;

        location / {
            proxy_pass http://127.0.0.1:8080;  # Apache backend vẫn HTTP
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_redirect off;
        }
    }
    ```
    - `listen 443 ssl`: lắng nghe HTTPS.
    - `ssl_certificate` và `ssl_certificate_key:`: đường dẫn tới file chứng chỉ công khai (.crt) của site và private key tương ứng (.key).
    - `add_header Content-Security-Policy - "upgrade-insecure-requests"`: chỉ thị trình duyệt tự động nâng request HTTP sang HTTPS, tránh mixed content (một số file tải bằng HTTP trên site HTTPS làm lỗi hiển thị).
    - `proxy_redirect off`: Nginx không tự động sửa URL trong header Location mà để WordPress quyết định redirect đúng theo cấu hình của nó. Tránh các lỗi liên  quan đến redirect khi dùng Nginx làm reverse proxy.
- Sau khi cấu hình, kiểm tra và restart lại Nginx để áp dụng.
    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```
#### Kiểm thử website wordpress.
- Truy cập web qua HTTP:
    ![alt text](./image-topic4/image-7.png)

- Truy cập web qua HTTPS:
    ![alt text](./image-topic4/image-6.png)

- Kiểm tra xem Nginx Reverse Proxy có hoạt động đúng không bằng cách tạm thời dừng Apache backend và truy cập lại trang web.
    ```bash
    sudo systemctl stop apache2
    ```
    ![alt text](./image-topic4/image-8.png)

    Kết quả trên hình cho thấy:
    - Nginx đang chạy và nhận request từ trình duyệt.
    - Nginx cố gắng gửi request tới backend (Apache) nhưng không kết nối được. Vì Apache đang dừng, nên Nginx trả lỗi 502 Bad Gateway.
    - Điều này chứng tỏ Nginx đang nhận request nhưng không thể forward tới Apache, nghĩa là proxy_pass đang hoạt động.
### Laravel
Cấu hình website Laravel về căn bản cũng tương tự như cấu hình WordPress.
#### Cấu hình Apache
- Tạo Virtual Host chứa website Laravel tại thư mục `/etc/apache2/sites-available/laravel.conf`
    ```bash
    <VirtualHost 127.0.0.1:8081>
        ServerName laravel.chiennguyen.vietnix.tech
        DocumentRoot /var/www/laravel/public
        <Directory /var/www/laravel/public>
            AllowOverride All
            Require all granted
        </Directory>
    </VirtualHost>
    ```
- Sau khi cấu hình, kích hoạt vhost và kiểm tra syntax, sau đó khởi động lại apache2
    ```bash
        sudo a2ensite laravel.conf
        sudo apache2ctl configtest
        sudo systemctl reload apache2
    ```
#### Cấu hình Nginx
- Cấu hình Nginx làm reverse proxy cho website Laravel chạy trên Apache backend. cấu hình 2 block server qua file `/etc/nginx/sites-available/laravel.chiennguyen.vietnix.tech` riêng biệt cho truy cập HTTP và HTTPS theo yêu cầu đề bài (Không cấu hình redirect từ HTTP -> HTTPS).
    ```bash
    server {
        listen 80;
        server_name laravel.chiennguyen.vietnix.tech;

        location / {
            proxy_pass http://127.0.0.1:8081;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }

    server {
        listen 443 ssl;
        server_name laravel.chiennguyen.vietnix.tech;

        ssl_certificate     /etc/nginx/ssl/laravel.crt;
        ssl_certificate_key /etc/nginx/ssl/laravel.key;

        root /var/www/laravel/public;
        index index.php index.html;

        add_header X-Frame-Options "SAMEORIGIN";
        add_header X-Content-Type-Options "nosniff";

        location / {
            proxy_pass http://127.0.0.1:8081;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```
- Sau khi cấu hình, kiểm tra và restart lại Nginx để áp dụng.
    ```bash
    sudo nginx -t
    sudo systemctl restart nginx
    ```
#### Kiểm thử website Laravel.
- Truy cập web qua HTTP:
![alt text](./image-topic4/image-12.png)
- Truy cập web qua HTTPS:
![alt text](./image-topic4/image-11.png)
- Kiểm tra xem Nginx Reverse Proxy có hoạt động đúng không bằng cách tạm thời dừng Apache backend và truy cập lại trang web.
    ```bash
    sudo systemctl stop apache2
    ```
    ![alt text](./image-topic4/image-9.png)
## Xây dựng Default vhost cho mọi domain và IP còn lại
### Cấu hình Apache
- Tạo Virtual Host chứa website default tại thư mục `/etc/apache2/sites-available/default.conf`
    ```bash
    <VirtualHost 127.0.0.1:8090>
        ServerName default
        DocumentRoot /var/www/test

        <Directory /var/www/test>
            AllowOverride All
            Require all granted
        DirectoryIndex test.html
        </Directory>

        ErrorLog ${APACHE_LOG_DIR}/default-error.log
        CustomLog ${APACHE_LOG_DIR}/default-access.log combined
    </VirtualHost>
    ```
    - `<VirtualHost 127.0.0.1:8090>`: Cấu hình máy chủ ảo Apache lắng nghe trên IP 127.0.0.1 và port 8090.
    - `ServerName default`: Đặt tên server là "default" (dùng khi hostname không khớp VirtualHost khác).
    - `DocumentRoot /var/www/test`: Thư mục gốc chứa file web của VirtualHost này.
    - `AllowOverride All`: Cho phép .htaccess ghi đè cấu hình Apache trong thư mục này.
    - `Require all granted`: Cho phép mọi client truy cập thư mục này.
    - `DirectoryIndex test.html`: File mặc định khi truy cập thư mục là test.html. Đây là một file html mẫu để biểu thị truy cập.
    - `ErrorLog ${APACHE_LOG_DIR}/default-error.log`: Đường dẫn file log lỗi của VirtualHost.
    - `CustomLog ${APACHE_LOG_DIR}/default-access.log combined`: Đường dẫn file log truy cập, dùng định dạng combined.

    - Sau khi cấu hình, kích hoạt vhost và kiểm tra syntax, sau đó khởi động lại apache2
    ```bash
        sudo a2ensite default.conf
        sudo apache2ctl configtest
        sudo systemctl reload apache2
    ```
### Cấu hình Nginx
- Trước khi cấu hình, ta phải vô hiệu hóa site default của Nginx để khi người dùng truy cập bằng IP hoặc domain chưa cấu hình, request không chạy site mặc định của Nginx mà sẽ được forward tới Apache VirtualHost đã cấu hình ở trên. Để làm điều đó ta hủy symlink site mặc định của Nginx:

```bash
sudo unlink /etc/nginx/sites-enabled/default
```
- Cấu hình Nginx làm reverse proxy cho website default trên Apache Backend. Cấu hình các block server qua file `/etc/nginx/sites-available/defaultproxy` riêng biệt cho truy cập HTTP và HTTPS theo yêu cầu đề bài (Không cấu hình redirect từ HTTP -> HTTPS).
    ```bash
    server {
        listen 80 default_server;
        server_name _;

        location / {
            proxy_pass http://127.0.0.1:8090;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```
    - `Listen 80 default_server`: Lắng nghe port 80 và đặt là default server (nếu request không khớp server_name nào khác)
    - `server_name _`: Dùng "_" làm wildcard, khớp tất cả các domain chưa được cấu hình và khớp với virtual host đằng sau.
    ```bash
    # HTTPS
    server {
        listen 443 ssl default_server;
        server_name _;

        ssl_certificate     /etc/nginx/ssl/nginx.crt;
        ssl_certificate_key /etc/nginx/ssl/nginx.key;

        location / {
            proxy_pass http://127.0.0.1:8090;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
    ```
    - `ssl_certificate` và `ssl_certificate_key:`: đường dẫn tới file chứng chỉ công khai (.crt) của site và private key tương ứng (.key). Ở đây thay vì sử dụng SSL của nhà cung cấp thì ta sẽ tự tạo 1 SSL tự cấp cho tất cả domain khác và IP.
- Sau khi cấu hình, tạo một file symlink vào thư mục `/etc/nginx/site-enabled/` rồi kiểm tra syntax và khởi động lại nginx để lưu cấu hình.
    ```bash
    sudo ln -s /etc/nginx/sites-available/defaultproxy /etc/nginx/sites-enabled/
    sudo nginx -t
    sudo systemctl restart nginx
    ```
### Kiểm thử
- Truy cập website qua HTTP bằng địa chỉ IP
![alt text](./image-topic4/image-13.png)
- Truy cập website qua HTTPS bằng địa chỉ IP
![alt text](./image-topic4/image-14.png)
- Truy cập website qua HTTP bằng một domain khác đã cấu hình phân giải về địa chỉ IP
![alt text](./image-topic4/image-15.png)
    -> Do SSL được cấu hình là SSL tự cấp, không phải SSL chính chủ nên trang web dù truy cập bằng HTTPS sẽ cảnh báo `connection not secure`