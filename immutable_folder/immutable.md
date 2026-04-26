# Immutable - HackTheOn Sejong 2026

**Category:** Pwn  
**File:** [Xem file Exploit tại đây](./for_user.ziptoken=eyJ1c2VyX2lkIjo0MzcsInRlYW1faWQiOjExOCwiZmlsZV9pZCI6MjQzfQ.aewbHw.beV5q9qBtaFUohjpYzDiJwfH8m4)
## Mô tả thử thách

Binary `immutable` yêu cầu người dùng nhập một chuỗi ký tự.  
Nếu thỏa mãn điều kiện nội bộ, chương trình sẽ in `"You win!"` và cấp shell. Ngược lại in `"You lose.."`.

## Chạy thử và phân tích file 
Ta cấp quyền chạy local cho file immutable và chạy thử xem chức năng của immutable hoạt động ra sao

![Local run](https://github.com/Writeup-Challenge-Le-Nam-Thang/Hacktheon/blob/8f442261caa630f812a34b5283946375813bede5/local.png)

Tiếp theo ta kiểm tra xem cấu hình của file này là gì bằng câu lệnh `file ./immutable`

![config](https://github.com/Writeup-Challenge-Le-Nam-Thang/Hacktheon/blob/8f442261caa630f812a34b5283946375813bede5/check.png)

Ta tiếp tục sử dụng công cụ ghidra để hỗ trợ dịch ngược giúp đọc file đễ dàng hơn



![Ghidra Decompile](https://github.com/Writeup-Challenge-Le-Nam-Thang/Hacktheon/blob/8f442261caa630f812a34b5283946375813bede5/ghidra.png)

### Phân tích lỗ hổng 
Hàm __isoc99_scanf("%s", local_98) đọc một chuỗi không giới hạn độ dài vào bộ đệm (buffer) trên stack. Do không giới hạn độ dài đầu vào, việc nhập vượt quá 128 bytes sẽ gây tràn bộ đệm và ghi đè lên các biến cục bộ nằm liền kề.

### Stack Layout
Dựa trên phân tích mã nguồn trong Ghidra, cấu trúc ngăn xếp (stack) được tổ chức như sau:

`rbp-0x90:` local_98 (128 bytes, buffer chứa dữ liệu nhập)

`rbp-0x10:` local_18 (4 bytes, biến kiểm tra điều kiện thắng)

`rbp-0x0C:` [4 bytes padding] - vùng đệm trống giữa biến và canary.

`rbp-0x08:` local_10 (8 bytes, stack canary)

Từ đó ta có thể tính được offset hay khoảng các từ `local_98` đến `local_18`: $0x90 - 0x10 = 0x80 = 128 \text{ bytes}$

---

## Exploit

Lại đến phần hay nhất rồi đó chính là viết payload để giải bài

```python
from pwn import *

# Khởi tạo kết nối
p = remote("3.37.44.62", 33201) #kết nối với cả địa chỉ của ban tổ chức
p = process("./immutable") #khi chạy local
payload = b"A" * 128 + p32(0xDEADBEEF)
p.sendlineafter(b"Give me a input: ", payload)
p.interactive()
```
Phân tích payload một chút 
`payload = b"A" * 128 + p32(0xDEADBEEF)`

`b"A" * 128` dùng để lấp đầy hoàn toàn bộ đệm local_98.
Tại sao là 128? Trong Ghidra, local_98 được khai báo là undefined1 `local_98[128]`.

Nó bắt đầu tại địa chỉ rbp-0x90 và kết thúc tại `rbp-0x10`.
$0x90 - 0x10 = 0x80$ (tức 128 trong hệ thập phân). 
Việc gửi 128 ký tự sẽ đưa con trỏ ghi dữ liệu đến ngay sát mép của biến mục tiêu `local_18`

``p32(0xDEADBEEF)`` dùng để ghi đè giá trị `0xDEADBEEF` vào biến ``local_18``

Tại sao dùng `p32`? Biến `local_18 `được định nghĩa là kiểu uint (unsigned integer), chiếm 4 bytes (32 bit) trên bộ nhớ. Hàm `p32()` của thư viện Pwntools sẽ đảm bảo đóng gói giá trị này đúng kích thước 4 bytes.

Little-endian: Vì tệp thực thi là `LSB` (Least Significant Byte), giá trị `0xDEADBEEF` phải được gửi dưới dạng đảo ngược các byte: ``\xef\xbe\xad\xde``. Hàm `p32` sẽ tự động xử lý việc này.

Chạy file và lấy flag:
![flag](https://github.com/Writeup-Challenge-Le-Nam-Thang/Hacktheon/blob/8f442261caa630f812a34b5283946375813bede5/flag.png)
