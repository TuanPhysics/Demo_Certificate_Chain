# Thực hành về chuỗi chứng chỉ số (Certificate Chain)

Các bước thực hành trên HĐH Ubuntu để hiểu về cơ chế hoạt động thực tế của chuỗi chứng chỉ số trong lớp bảo mật TLS khi kết nối với hai broker phổ biến là EMQX và HiveMQ.

## Chuẩn bị

- Đăng ký tài khoản trên EMQX và HiveMQ: xem hình chụp màn hình trong slide về MQTT
- Khởi tạo các broker miễn phí trên máy chủ của hai nền tảng này -> thu được địa chỉ endpoint của broker vừa khởi tạo. Lưu ý: các endpoint (hay có chỗ gọi là URL) của broker của bạn là khác địa chỉ của tôi minh họa trong ví dụ dưới đây.
- Các demo dưới đều làm trên công cụ openssl có sẵn trên Ubuntu. Nếu bạn trên Windows có thể dùng qua WSL hoặc trên `Git bash` (đã cài khi bạn cài Git). 

## Xem chuỗi chứng chỉ của dịch vụ EMQX
- Chạy lệnh `openssl s_client -showcerts -connect xxx12345.ala.asia-southeast1.emqxsl.com:8883`. Thay `xxx12345` bằng địa chỉ URL tương ứng của bạn khi khởi tạo trên EMQX console.
- Đầu ra chi tiết của lệnh này được cho trong file `emqxsl_chain.txt` trong thư mục này. 
- Đoạn đầu có nội dung:
    ```bash
    CONNECTED(00000003)
    depth=2 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = DigiCert Global Root G2
    verify return:1
    depth=1 C = US, O = DigiCert Inc, OU = www.digicert.com, CN = Encryption Everywhere DV TLS CA - G2
    verify return:1
    depth=0 CN = *.ala.asia-southeast1.emqxsl.com
    verify return:1
    ---
    ```

## Phân tích chuỗi chứng chỉ thu được khi kết nối với server EMQX

- Lệnh s_client của openssl khi kết nối với broker trên EMQX trả về một chuỗi các chứng chỉ (từ server). Trong đó có các `độ sâu` (depth) khác nhau. 
- `depth=2` là độ sâu nhất, là chứng chỉ ở tầng trên cùng được cấp bởi một Root CA gọi là DigiCert.
- Root CA này sẽ ký xác nhận cho một CA trung gian ở `depth=1`, ở đây chinh slà tổ chức có tên Encryption Everywhere. Như bạn thấy ở nội dung trên.
- Ở tầng dưới cùng `depth=0` chính là chứng chỉ của một nhóm các subdomain có dạng `*.ala.asia-southeast1.emqxsl.com` của EMQX. Đây chính là nhóm domain chứa URL đến broker của các bạn vừa tạo ra trên server. Chứng chỉ này được ký bởi tổ chức CA trung gian (Encryption Everywhere) ở trên.

## Xác nhận chứng chỉ thu được là "thực"
(hay còn gọi là xác thực chứng chỉ bằng chữ ký số)

Nhắc lại một chút kiến thức về vai trò của chứng chỉ số trong TLS: nó là một bằng chứng xác nhận rằng tôi chính là cái domain này, và đây là khóa công khai của tôi, _nếu bạn tin_ thì dùng khóa này để mã khóa thông tin trước khi gửi cho tôi.

Nhưng làm sao để tin nó đúng là nó, vì có thể bị giả mạo domain trên đường truyền chẳng hạn? Cơ chế bảo vệ việc này như sau:

- Chứng chỉ số của trang đó (chứng chỉ lớp `depth=0` ở ví dụ trên) nó có bao gồm một chữ ký số xác thực của tổ chức CA bên trên (lớp `depth=1` trong ví dụ đang phân tích). Lớp trung gian lại cần xác thực (ký chứng chỉ) bởi một tổ chức đáng tin cuối cùng gọi là Root CA. 

## Vậy thì làm sao để biết Root CA có đáng tin không? 

Câu trả lời như sau:
- Ta có thể minh họa việc này bằng chính máy tính Ubuntu của tôi. Hệ điều hành này được cập nhật thường xuyên và có một file quan trọng ở `/etc/ssl/certs/ca-certificates.crt` chứa chứng chỉ của tất cả các tổ chức root CA đáng tin ở thời điểm hiện tại. Trách nhiệm duy trì và cập nhật danh sách này thuộc về hệ điều hành. Như ta mở file đó ra thì thấy: hiện tại trong danh sách này có khoảng hơn 50 Root CA trên toàn thế giới thôi. Trong đó ta tìm thấy "DigiCert". (tôi sẽ show màn hình ở bước này)
- Khi kết nối đến trang bên trên (từ một công cụ của hệ điều hành như openssl) thì nó sẽ sử dụng danh sách chứng chỉ Root CA đã lưu sẵn - mỗi cái có một public key để xác thực chữ ký số của CA tương ứng. Hệ điều hành sẽ sử dụng public key nó lưu của Root CA cần thiết để xác thực các chứng chỉ bên dưới từ server mà nó kết nối vào. Nếu đúng thì server đó đúng là có địa chỉ như ta mong đợi, và cái khóa công khai của nó ở chứng chỉ cuối cùng đúng là của nó (đã đăng ký với CA trung gian).

## Vậy cái file `emqxsl-ca.crt` mà EMQX cung cấp trên trang console của họ là như thế nào?

- File này là chứng chỉ số Root CA dùng để nạp vào con chip muốn làm client kết nối đến broker họ cung cấp qua lớp bảo mật TLS.
- Liệu file này có đáng tin? 
    + Cơ bản là tin được, vì mình vào bằng browser của mình không lẽ họ tự giả mạo chính họ. 
    + Nhưng ta có thể check trên `https://knowledge.digicert.com/general-information/digicert-trusted-root-authority-certificates#roots` và thấy một chứng chỉ có nội dung y hệt. 
    + Thậm chí tìm trong `/etc/ssl/certs/ca-certificates.crt` cũng thấy một chứng chỉ như vậy luôn. 
    + Vậy đây đúng là một chứng chỉ Root CA thực của Digicert.

- ESP32 làm gì với file này?
    + Thư viện Secured Client trên chip sẽ phải dùng nó để xác thực khi thiết lập kết nối TLS với broker (qua cổng 8883). Cơ bản thì cơ chế như trên máy tính thôi: dùng khóa công khai trong CA Cert (trong file kia có chứa) để xác thực chứng chỉ trung gian mà server cung cấp. Sau đó dùng public key trong chứng chỉ trung gian đã xác thực để xác thực tiếp chứng chỉ cuối cùng chứa khóa công khai của server. 
    + ESP32 bộ nhớ nhỏ và tính toán yếu cho nên không thể lưu danh sách Root CA dài như trên Ubuntu được.

- Có thể nháy đúp vào file này để hiển thị nội dung dễ đọc (hệ điều hành cung cấp tiện ích này). 
- Cũng có thể dùng `openssl x509 -in emqxsl-ca.crt -text -noout > emqxsl-ca.txt` để chuyển nội dung đã được mã hóa bằng base64 (*) thành thông tin dễ đọc ở file text.
- (*base64) đây là loại mã hóa chuyển nhị phân thành text để truyền đi qua internet thôi, không phải để giấu thông tin

## Bài tập luyện tập 

- Lặp lại tất cả các bước kể trên với địa chỉ broker mà bạn đăng ký được trên EMQX 
- Lặp lại tất cả các bước kể trên với địa chỉ broker mà bạn đăng ký được trên HiveMQ
- Bạn sẽ thấy HiveMQ hơi khác một chút. Có lẽ tôi sẽ làm cái này cho các bạn xem trước.

## Câu hỏi mở 

- Nếu các bạn dùng mosquitto mqtt broker trên server riêng của các bạn (ví dụ: raspberry Pi) thì việc kết nối TLS (secured client) có cần thiết hay không?
- Nếu có, thì ta phải làm thế nào?
