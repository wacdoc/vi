# Xây dựng máy chủ gửi thư SMTP của riêng bạn

## lời mở đầu

SMTP có thể trực tiếp mua dịch vụ từ các nhà cung cấp đám mây, chẳng hạn như:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Đẩy email đám mây Ali](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Bạn cũng có thể xây dựng máy chủ thư của riêng mình - gửi không giới hạn, chi phí tổng thể thấp.

Dưới đây, chúng tôi trình bày từng bước cách xây dựng máy chủ thư của riêng mình.

## Lựa chọn máy chủ

Máy chủ SMTP tự lưu trữ yêu cầu IP công khai với các cổng 25, 456 và 587 đang mở.

Các đám mây công cộng thường được sử dụng đã chặn các cổng này theo mặc định và có thể mở chúng bằng cách đưa ra lệnh sản xuất, nhưng dù sao thì nó cũng rất rắc rối.

Tôi khuyên bạn nên mua từ một máy chủ có các cổng mở này và hỗ trợ thiết lập tên miền đảo ngược.

Ở đây, tôi khuyên dùng [Contabo](https://contabo.com) .

Contabo là nhà cung cấp hosting có trụ sở tại Munich, Đức, được thành lập vào năm 2003 với giá cả rất cạnh tranh.

Nếu bạn chọn Euro làm đơn vị tiền tệ mua hàng, giá sẽ rẻ hơn (máy chủ có bộ nhớ 8GB và 4 CPU có giá khoảng 529 nhân dân tệ mỗi năm và miễn phí phí ​​cài đặt ban đầu trong một năm).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Khi đặt hàng ghi chú `prefer AMD` , và máy chủ có CPU AMD sẽ có hiệu năng tốt hơn.

Trong phần sau, tôi sẽ lấy VPS của Contabo làm ví dụ để minh họa cách xây dựng máy chủ thư của riêng bạn.

## Cấu hình hệ thống Ubuntu

Hệ điều hành ở đây là Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Nếu máy chủ trên ssh hiển thị `Welcome to TinyCore 13!` (như hình bên dưới), tức là hệ thống chưa được cài đặt. Vui lòng ngắt kết nối ssh và đợi trong vài phút để đăng nhập lại.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Khi `Welcome to Ubuntu 22.04.1 LTS` xuất hiện, quá trình khởi tạo đã hoàn tất và bạn có thể tiếp tục với các bước sau.

### [Tùy chọn] Khởi tạo môi trường phát triển

Bước này là tùy chọn.

Để thuận tiện, tôi đặt cài đặt và cấu hình hệ thống của phần mềm ubuntu trong [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Chạy lệnh sau để cài đặt bằng một cú nhấp chuột.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Người dùng Trung Quốc, vui lòng sử dụng lệnh sau để thay thế và ngôn ngữ, múi giờ, v.v. sẽ được đặt tự động.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo kích hoạt IPV6

Bật IPV6 để SMTP cũng có thể gửi email có địa chỉ IPV6.

chỉnh sửa `/etc/sysctl.conf`

Sửa đổi hoặc thêm các dòng sau

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Theo dõi [hướng dẫn contabo: Thêm kết nối IPv6 vào máy chủ của bạn](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Chỉnh sửa `/etc/netplan/01-netcfg.yaml` , thêm vài dòng như hình bên dưới (file cấu hình mặc định của Contabo VPS đã có sẵn các dòng này, chỉ cần bỏ ghi chú).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Sau đó `netplan apply` để làm cho cấu hình đã sửa đổi có hiệu lực.

Sau khi cấu hình thành công, bạn có thể sử dụng `curl 6.ipw.cn` để xem địa chỉ ipv6 của mạng bên ngoài.

## Sao chép ops kho lưu trữ cấu hình

```
git clone https://github.com/wactax/ops.soft.git
```

## Tạo chứng chỉ SSL miễn phí cho tên miền của bạn

Gửi thư yêu cầu chứng chỉ SSL để mã hóa và ký tên.

Chúng tôi sử dụng [acme.sh](https://github.com/acmesh-official/acme.sh) để tạo chứng chỉ.

acme.sh là một công cụ ký chứng chỉ tự động mã nguồn mở,

Nhập kho cấu hình ops.soft, chạy `./ssl.sh` và thư mục `conf` sẽ được tạo trong **thư mục trên** .

Tìm nhà cung cấp DNS của bạn từ [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , chỉnh sửa `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Sau đó chạy `./ssl.sh 123.com` để tạo chứng chỉ `123.com` và `*.123.com` cho tên miền của bạn.

Lần chạy đầu tiên sẽ tự động cài đặt [acme.sh](https://github.com/acmesh-official/acme.sh) và thêm tác vụ đã lên lịch để gia hạn tự động. Bạn có thể thấy `crontab -l` , có một dòng như sau.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Đường dẫn cho chứng chỉ được tạo giống như `/mnt/www/.acme.sh/123.com_ecc。`

Gia hạn chứng chỉ sẽ gọi tập lệnh `conf/reload/123.com.sh` , chỉnh sửa tập lệnh này, bạn có thể thêm các lệnh như `nginx -s reload` để làm mới bộ đệm chứng chỉ của các ứng dụng liên quan.

## Xây dựng máy chủ SMTP với chasquid

[chasquid](https://github.com/albertito/chasquid) là một máy chủ SMTP mã nguồn mở được viết bằng ngôn ngữ Go.

Để thay thế cho các chương trình máy chủ thư cũ như Postfix và Sendmail, chasquid đơn giản và dễ sử dụng hơn, đồng thời cũng dễ dàng hơn cho việc phát triển thứ cấp.

Chạy `./chasquid/init.sh 123.com` sẽ được cài đặt tự động bằng một cú nhấp chuột (thay thế 123.com bằng tên miền gửi của bạn).

## Định cấu hình chữ ký email DKIM

DKIM được sử dụng để gửi chữ ký email nhằm ngăn chặn các bức thư bị coi là thư rác.

Sau khi lệnh chạy thành công, bạn sẽ được nhắc thiết lập bản ghi DKIM (như hình bên dưới).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Chỉ cần thêm bản ghi TXT vào DNS của bạn (như hình bên dưới).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Xem trạng thái và nhật ký dịch vụ

 `systemctl status chasquid` Xem trạng thái dịch vụ.

Tình trạng hoạt động bình thường như hình bên dưới

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` hoặc `journalctl -xeu chasquid` có thể xem nhật ký lỗi.

## Cấu hình tên miền ngược

Đảo ngược tên miền là cho phép phân giải địa chỉ IP sang tên miền tương ứng.

Đặt tên miền đảo ngược có thể ngăn email bị xác định là thư rác.

Khi nhận được thư, máy chủ nhận sẽ thực hiện phân tích tên miền ngược trên địa chỉ IP của máy chủ gửi để xác nhận xem máy chủ gửi có tên miền ngược hợp lệ hay không.

Nếu máy chủ gửi không có tên miền đảo ngược hoặc nếu tên miền đảo ngược không khớp với địa chỉ IP của máy chủ gửi, máy chủ nhận có thể nhận ra email là thư rác hoặc từ chối email đó.

Truy cập [https://my.contabo.com/rdns](https://my.contabo.com/rdns) và cấu hình như hình bên dưới

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Sau khi đặt tên miền ngược nhớ cấu hình chuyển tiếp độ phân giải của tên miền ipv4 và ipv6 về máy chủ.

## Chỉnh sửa tên máy chủ của chasquid.conf

Sửa đổi `conf/chasquid/chasquid.conf` thành giá trị của tên miền đảo ngược.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Sau đó chạy `systemctl restart chasquid` để khởi động lại dịch vụ.

## Sao lưu conf vào kho git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Ví dụ mình backup thư mục conf vào github process của mình như sau

Tạo kho riêng trước

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Nhập thư mục conf và gửi đến kho

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## thêm người gửi

chạy

```
chasquid-util user-add i@wac.tax
```

Có thể thêm người gửi

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Xác minh rằng mật khẩu được đặt chính xác

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Sau khi thêm người dùng, `chasquid/domains/wac.tax/users` sẽ được cập nhật, nhớ nộp vào kho.

## DNS thêm bản ghi SPF

SPF (Sender Policy Framework) là một công nghệ xác minh email được sử dụng để ngăn chặn gian lận email.

Nó xác minh danh tính của người gửi thư bằng cách kiểm tra xem địa chỉ IP của người gửi có khớp với bản ghi DNS của tên miền mà nó tuyên bố hay không, ngăn chặn những kẻ lừa đảo gửi email không có thật.

Việc thêm các bản ghi SPF có thể ngăn các email bị xác định là thư rác nhiều nhất có thể.

Nếu máy chủ tên miền của bạn không hỗ trợ loại SPF, chỉ cần thêm bản ghi loại TXT.

Ví dụ: SPF của `wac.tax` như sau

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF cho `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Lưu ý rằng tôi đã `include:_spf.google.com` ở đây, điều này là do tôi sẽ định cấu hình `i@wac.tax` làm địa chỉ gửi trong hộp thư Google sau này.

## Cấu hình DNS DMARC

DMARC là tên viết tắt của (Xác thực, báo cáo & tuân thủ thông báo dựa trên miền).

Nó được sử dụng để nắm bắt các lần trả lại SPF (có thể do lỗi cấu hình hoặc ai đó đang giả danh bạn để gửi thư rác).

Thêm bản ghi TXT `_dmarc` ,

Nội dung như sau

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Ý nghĩa của từng thông số như sau

### p (Chính sách)

Cho biết cách xử lý các email không xác minh được SPF (Khung chính sách người gửi) hoặc DKIM (Thư được xác định bằng khóa miền). Tham số p có thể được đặt thành một trong ba giá trị:

* none: Không có hành động nào được thực hiện, chỉ có kết quả xác minh được gửi lại cho người gửi thông qua cơ chế báo cáo email.
* Kiểm dịch: Đặt thư chưa vượt qua xác minh vào thư mục thư rác, nhưng sẽ không từ chối thư trực tiếp.
* từ chối: Từ chối trực tiếp các email không xác minh được.

### fo (Tùy chọn lỗi)

Chỉ định lượng thông tin được trả về bởi cơ chế báo cáo. Nó có thể được đặt thành một trong các giá trị sau:

* 0: Báo cáo kết quả xác thực cho tất cả thư
* 1: Chỉ báo cáo những tin nhắn không xác minh được
* d: Chỉ báo lỗi xác minh tên miền
* s: chỉ báo cáo lỗi xác minh SPF
* l: Chỉ báo lỗi xác minh DKIM

### rua & rua

* rua (URI báo cáo cho báo cáo tổng hợp): Địa chỉ email để nhận báo cáo tổng hợp
* ruf (URI báo cáo cho báo cáo pháp y): địa chỉ email để nhận báo cáo chi tiết

## Thêm bản ghi MX để chuyển tiếp email tới Gmail

Do không tìm được hộp thư công ty miễn phí hỗ trợ địa chỉ phổ dụng (Catch-All, có thể nhận bất kỳ email nào gửi đến tên miền này, không hạn chế tiền tố), nên tôi đã sử dụng chasquid để chuyển tiếp tất cả email đến hộp thư Gmail của mình.

**Nếu bạn có hộp thư doanh nghiệp trả phí của riêng mình, vui lòng không sửa đổi MX và bỏ qua bước này.**

Chỉnh sửa `conf/chasquid/domains/wac.tax/aliases` , đặt hộp thư chuyển tiếp

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` cho biết tất cả các email, `i` là tiền tố địa chỉ email của người gửi đã tạo ở trên. Để chuyển tiếp thư, mỗi người dùng cần thêm một dòng.

Sau đó thêm bản ghi MX (tôi trỏ trực tiếp đến địa chỉ của tên miền ngược ở đây, như thể hiện trong dòng đầu tiên trong hình bên dưới).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Sau khi cấu hình xong, bạn có thể sử dụng các địa chỉ email khác để gửi email tới `i@wac.tax` và `any123@wac.tax` để xem có nhận được email trong Gmail hay không.

Nếu không, hãy kiểm tra nhật ký chasquid ( `grep chasquid /var/log/syslog` ).

## Gửi email tới i@wac.tax bằng Gmail

Sau khi Google Mail nhận được thư, tôi tự nhiên hy vọng sẽ trả lời bằng `i@wac.tax` thay vì i.wac.tax@gmail.com.

Truy cập [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) và nhấp vào "Thêm địa chỉ email khác".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Sau đó, nhập mã xác minh nhận được từ email đã được chuyển tiếp đến.

Cuối cùng, nó có thể được đặt làm địa chỉ người gửi mặc định (cùng với tùy chọn trả lời bằng cùng một địa chỉ).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Như vậy là chúng ta đã hoàn thành việc thiết lập SMTP mail server, đồng thời sử dụng Google Mail để gửi và nhận email.

## Gửi mail kiểm tra cấu hình có thành công không

Nhập `ops/chasquid`

Chạy `direnv allow` cài đặt các phụ thuộc (direnv đã được cài đặt trong quá trình khởi tạo một khóa trước đó và một hook đã được thêm vào trình bao)

sau đó chạy

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Ý nghĩa của các thông số như sau

* người dùng: tên người dùng SMTP
* vượt qua: mật khẩu SMTP
* đến: người nhận

Bạn có thể gửi một email kiểm tra.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Nên sử dụng Gmail để nhận email test kiểm tra cấu hình có thành công hay không.

### Mã hóa tiêu chuẩn TLS

Như hình bên dưới có ổ khóa nhỏ này tức là đã kích hoạt chứng chỉ SSL thành công.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Sau đó nhấp vào "Hiển thị Email gốc"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Như hình bên dưới, trang mail gốc Gmail hiển thị DKIM, tức là cấu hình DKIM thành công.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Kiểm tra Đã nhận trong tiêu đề của email ban đầu và bạn có thể thấy rằng địa chỉ người gửi là IPV6, điều đó có nghĩa là IPV6 cũng được định cấu hình thành công.
