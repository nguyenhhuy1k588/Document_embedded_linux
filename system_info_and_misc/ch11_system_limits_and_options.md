## Chương 11
# Giới Hạn Hệ Thống và Các Tùy Chọn (System Limits and Options)

Mỗi triển khai UNIX đặt ra giới hạn cho các tính năng và tài nguyên hệ thống khác nhau, và cung cấp — hoặc không cung cấp — các tùy chọn được định nghĩa trong nhiều chuẩn khác nhau. Các ví dụ bao gồm:

- Một process có thể mở bao nhiêu file cùng lúc?
- Hệ thống có hỗ trợ realtime signal không?
- Giá trị lớn nhất có thể lưu trong biến kiểu `int` là bao nhiêu?
- Danh sách đối số của một chương trình có thể lớn đến mức nào?
- Độ dài tối đa của một pathname là bao nhiêu?

Tuy có thể hard-code các giả định về giới hạn và tùy chọn vào ứng dụng, điều này làm giảm tính portability vì giới hạn và tùy chọn có thể thay đổi:

- **Giữa các triển khai UNIX**: Dù giới hạn và tùy chọn có thể cố định trên từng triển khai, chúng có thể khác nhau giữa các triển khai UNIX khác nhau. Giá trị lớn nhất có thể lưu trong `int` là một ví dụ về giới hạn như vậy.
- **Tại runtime trên một triển khai cụ thể**: Ví dụ kernel có thể được cấu hình lại để thay đổi giới hạn. Hoặc ứng dụng có thể được biên dịch trên một hệ thống nhưng chạy trên hệ thống khác với giới hạn và tùy chọn khác.
- **Từ file system này sang file system khác**: Ví dụ, các file system System V truyền thống cho phép tên file tối đa 14 byte, trong khi các file system BSD truyền thống và hầu hết file system Linux gốc cho phép tên file tối đa 255 byte.

Vì giới hạn và tùy chọn hệ thống ảnh hưởng đến những gì ứng dụng có thể làm, ứng dụng portable cần các cách để xác định giá trị giới hạn và các tùy chọn nào được hỗ trợ. Các chuẩn ngôn ngữ lập trình C và SUSv3 cung cấp hai con đường chính để ứng dụng lấy thông tin đó:

- Một số giới hạn và tùy chọn có thể xác định tại compile time. Ví dụ, giá trị lớn nhất của `int` được xác định bởi kiến trúc phần cứng và các lựa chọn thiết kế compiler. Các giới hạn như vậy có thể được ghi trong header file.
- Các giới hạn và tùy chọn khác có thể thay đổi tại runtime. Đối với những trường hợp này, SUSv3 định nghĩa ba hàm — `sysconf()`, `pathconf()`, và `fpathconf()` — mà ứng dụng có thể gọi để kiểm tra các giới hạn và tùy chọn triển khai.

SUSv3 quy định một tập giới hạn mà triển khai tuân thủ có thể áp đặt, cũng như một tập tùy chọn, mỗi tùy chọn có thể có hoặc không được hỗ trợ bởi một hệ thống cụ thể. Chúng ta mô tả một số giới hạn và tùy chọn này trong chương này, và mô tả các giới hạn khác tại các điểm liên quan trong các chương sau.

---

## 11.1 Giới Hạn Hệ Thống

Đối với mỗi giới hạn mà nó quy định, SUSv3 yêu cầu tất cả các triển khai phải hỗ trợ một giá trị tối thiểu cho giới hạn đó. Trong hầu hết các trường hợp, giá trị tối thiểu này được định nghĩa là một hằng số trong `<limits.h>` với tên có tiền tố `_POSIX_`, và (thường) chứa chuỗi `_MAX`; do đó tên có dạng `_POSIX_XXX_MAX`.

Nếu ứng dụng giới hạn bản thân ở các giá trị tối thiểu mà SUSv3 yêu cầu cho mỗi giới hạn, nó sẽ portable trên tất cả các triển khai theo chuẩn. Tuy nhiên, làm vậy ngăn ứng dụng tận dụng các triển khai cung cấp giới hạn cao hơn. Vì lý do này, thường tốt hơn là xác định giới hạn trên một hệ thống cụ thể bằng `<limits.h>`, `sysconf()`, hoặc `pathconf()`.

> Việc dùng chuỗi `_MAX` trong tên giới hạn do SUSv3 định nghĩa có vẻ gây nhầm lẫn, vì chúng được mô tả là giá trị tối thiểu. Lý do đặt tên trở nên rõ ràng khi ta xem xét rằng mỗi hằng số này định nghĩa giới hạn trên của một tài nguyên hoặc tính năng, và các chuẩn đang nói rằng giới hạn trên này phải có giá trị tối thiểu nhất định.

> Trong một số trường hợp, giá trị tối đa được cung cấp cho một giới hạn, và các giá trị này có tên chứa chuỗi `_MIN`. Đối với các hằng số này, điều ngược lại đúng: chúng đại diện cho giới hạn dưới của một tài nguyên, và các chuẩn đang nói rằng trên một triển khai tuân thủ, giới hạn dưới này không thể lớn hơn một giá trị nhất định. Ví dụ, giới hạn `FLT_MIN` (1E-37) định nghĩa giá trị lớn nhất mà một triển khai có thể đặt cho số floating-point nhỏ nhất có thể biểu diễn, và tất cả các triển khai tuân thủ sẽ có thể biểu diễn số floating-point ít nhất nhỏ đến mức đó.

Mỗi giới hạn có một tên tương ứng với tên giá trị tối thiểu mô tả ở trên, nhưng không có tiền tố `_POSIX_`. Một triển khai có thể định nghĩa hằng số với tên này trong `<limits.h>` để chỉ ra giới hạn tương ứng cho triển khai đó. Nếu được định nghĩa, giới hạn này sẽ luôn ít nhất bằng kích thước của giá trị tối thiểu mô tả ở trên (tức là `XXX_MAX >= _POSIX_XXX_MAX`).

SUSv3 chia các giới hạn mà nó quy định thành ba loại: runtime invariant values, pathname variable values, và runtime increasable values. Trong các đoạn sau, chúng ta mô tả các loại này và cung cấp một số ví dụ.

### Runtime invariant values (có thể không xác định)

Runtime invariant value là giới hạn có giá trị, nếu được định nghĩa trong `<limits.h>`, là cố định cho triển khai. Tuy nhiên, giá trị có thể không xác định (có thể vì nó phụ thuộc vào không gian bộ nhớ có sẵn), và do đó bị bỏ qua khỏi `<limits.h>`. Trong trường hợp này (và ngay cả khi giới hạn cũng được định nghĩa trong `<limits.h>`), ứng dụng có thể dùng `sysconf()` để xác định giá trị tại runtime.

Giới hạn `MQ_PRIO_MAX` là ví dụ về runtime invariant value. Như đã nói trong Mục 52.5.1, có giới hạn về độ ưu tiên cho message trong POSIX message queue. SUSv3 định nghĩa hằng số `_POSIX_MQ_PRIO_MAX` với giá trị 32, là giá trị tối thiểu mà tất cả các triển khai tuân thủ phải cung cấp cho giới hạn này. Điều này có nghĩa ta có thể chắc chắn tất cả các triển khai tuân thủ sẽ cho phép mức ưu tiên từ 0 đến ít nhất 31 cho message. Một triển khai UNIX có thể đặt giới hạn cao hơn, định nghĩa hằng số `MQ_PRIO_MAX` trong `<limits.h>` với giá trị giới hạn của nó. Ví dụ, trên Linux, `MQ_PRIO_MAX` được định nghĩa với giá trị 32,768. Giá trị này cũng có thể được xác định tại runtime bằng lời gọi sau:

```c
lim = sysconf(_SC_MQ_PRIO_MAX);
```

### Pathname variable values

Pathname variable values là các giới hạn liên quan đến pathname (file, directory, terminal, v.v.). Mỗi giới hạn có thể cố định cho triển khai hoặc có thể thay đổi từ file system này sang file system khác. Trong các trường hợp giới hạn có thể thay đổi tùy theo pathname, ứng dụng có thể xác định giá trị của nó bằng `pathconf()` hoặc `fpathconf()`.

Giới hạn `NAME_MAX` là ví dụ về pathname variable value. Giới hạn này định nghĩa kích thước tối đa cho tên file trên một file system cụ thể. SUSv3 định nghĩa hằng số `_POSIX_NAME_MAX` với giá trị 14 (giới hạn file system System V cũ), là giá trị tối thiểu mà triển khai phải cho phép. Một triển khai có thể định nghĩa `NAME_MAX` với giới hạn cao hơn và/hoặc cung cấp thông tin về file system cụ thể qua lời gọi có dạng:

```c
lim = pathconf(directory_path, _PC_NAME_MAX)
```

`directory_path` là pathname cho một directory trên file system cần quan tâm.

### Runtime increasable values

Runtime increasable value là giới hạn có giá trị tối thiểu cố định cho một triển khai cụ thể, và tất cả các hệ thống chạy triển khai đó sẽ cung cấp ít nhất giá trị tối thiểu này. Tuy nhiên, một hệ thống cụ thể có thể tăng giới hạn này tại runtime, và ứng dụng có thể tìm giá trị thực tế được hỗ trợ trên hệ thống bằng `sysconf()`.

Ví dụ về runtime increasable value là `NGROUPS_MAX`, định nghĩa số lượng supplementary group ID tối đa đồng thời của một process (Mục 9.6). SUSv3 định nghĩa giá trị tối thiểu tương ứng, `_POSIX_NGROUPS_MAX`, với giá trị 8. Tại runtime, ứng dụng có thể lấy giới hạn bằng lời gọi `sysconf(_SC_NGROUPS_MAX)`.

### Tóm tắt một số giới hạn SUSv3

Bảng 11-1 liệt kê một số giới hạn do SUSv3 định nghĩa có liên quan đến cuốn sách này (các giới hạn khác được giới thiệu trong các chương sau).

**Bảng 11-1:** Một số giới hạn SUSv3

| Tên giới hạn (`<limits.h>`) | Giá trị min | Tên `sysconf()` / `pathconf()` (`<unistd.h>`) | Mô tả |
|-----------------------------|-------------|-----------------------------------------------|-------|
| `ARG_MAX` | 4096 | `_SC_ARG_MAX` | Số byte tối đa cho các đối số (argv) cộng môi trường (environ) có thể truyền vào `exec()` (Mục 6.7 và 27.2.3) |
| none | none | `_SC_CLK_TCK` | Đơn vị đo thời gian cho `times()` |
| `LOGIN_NAME_MAX` | 9 | `_SC_LOGIN_NAME_MAX` | Kích thước tối đa của login name (gồm null byte kết thúc) |
| `OPEN_MAX` | 20 | `_SC_OPEN_MAX` | Số file descriptor tối đa một process có thể mở cùng lúc, bằng một cộng số descriptor lớn nhất có thể dùng (Mục 36.2) |
| `NGROUPS_MAX` | 8 | `_SC_NGROUPS_MAX` | Số supplementary group tối đa mà một process có thể là thành viên (Mục 9.7.3) |
| none | 1 | `_SC_PAGESIZE` | Kích thước của một virtual memory page (`_SC_PAGE_SIZE` là synonym) |
| `RTSIG_MAX` | 8 | `_SC_RTSIG_MAX` | Số realtime signal riêng biệt tối đa (Mục 22.8) |
| `SIGQUEUE_MAX` | 32 | `_SC_SIGQUEUE_MAX` | Số realtime signal xếp hàng tối đa (Mục 22.8) |
| `STREAM_MAX` | 8 | `_SC_STREAM_MAX` | Số stdio stream có thể mở cùng lúc tối đa |
| `NAME_MAX` | 14 | `_PC_NAME_MAX` | Số byte tối đa trong tên file, không tính null byte kết thúc |
| `PATH_MAX` | 256 | `_PC_PATH_MAX` | Số byte tối đa trong một pathname, kể cả null byte kết thúc |
| `PIPE_BUF` | 512 | `_PC_PIPE_BUF` | Số byte tối đa có thể ghi atomic vào pipe hoặc FIFO (Mục 44.1) |

Cột đầu tiên của Bảng 11-1 cho tên giới hạn, có thể được định nghĩa là hằng số trong `<limits.h>` để chỉ ra giới hạn cho một triển khai cụ thể. Cột thứ hai là giá trị tối thiểu do SUSv3 định nghĩa cho giới hạn (cũng được định nghĩa trong `<limits.h>`). Trong hầu hết các trường hợp, mỗi giá trị tối thiểu được định nghĩa là hằng số có tiền tố `_POSIX_`. Ví dụ, hằng số `_POSIX_RTSIG_MAX` (định nghĩa với giá trị 8) quy định giá trị tối thiểu SUSv3 yêu cầu tương ứng với hằng số triển khai `RTSIG_MAX`. Cột thứ ba chỉ định tên hằng số có thể được truyền tại runtime cho `sysconf()` hoặc `pathconf()` để lấy giới hạn triển khai. Các hằng số bắt đầu bằng `_SC_` dùng với `sysconf()`; những hằng số bắt đầu bằng `_PC_` dùng với `pathconf()` và `fpathconf()`.

Lưu ý các thông tin bổ sung sau so với Bảng 11-1:

- Hàm `getdtablesize()` là cách thay thế lỗi thời để xác định giới hạn file descriptor của process (`OPEN_MAX`). Hàm này được quy định trong SUSv2 (đánh dấu LEGACY), nhưng bị loại bỏ trong SUSv3.
- Hàm `getpagesize()` là cách thay thế lỗi thời để xác định kích thước page hệ thống (`_SC_PAGESIZE`). Hàm này được quy định trong SUSv2 (đánh dấu LEGACY), nhưng bị loại bỏ trong SUSv3.
- Hằng số `FOPEN_MAX`, định nghĩa trong `<stdio.h>`, là synonym với `STREAM_MAX`.
- `NAME_MAX` không tính null byte kết thúc, trong khi `PATH_MAX` có tính. Sự không nhất quán này sửa một không nhất quán trước đó trong chuẩn POSIX.1 khiến không rõ `PATH_MAX` có bao gồm null byte kết thúc hay không. Định nghĩa `PATH_MAX` bao gồm terminator có nghĩa là các ứng dụng chỉ cấp phát đúng `PATH_MAX` byte cho pathname vẫn tuân thủ chuẩn.

### Xác định giới hạn và tùy chọn từ shell: getconf

Từ shell, ta có thể dùng lệnh `getconf` để lấy các giới hạn và tùy chọn được triển khai bởi một UNIX cụ thể. Dạng chung của lệnh này như sau:

```
$ getconf variable-name [ pathname ]
```

`variable-name` xác định giới hạn cần quan tâm và là một trong các tên giới hạn chuẩn SUSv3, như `ARG_MAX` hoặc `NAME_MAX`. Khi giới hạn liên quan đến pathname, ta phải chỉ định pathname làm đối số thứ hai, như trong ví dụ thứ hai sau:

```
$ getconf ARG_MAX
131072
$ getconf NAME_MAX /boot
255
```

---

## 11.2 Lấy Giới Hạn Hệ Thống (và Tùy Chọn) tại Runtime

Hàm `sysconf()` cho phép ứng dụng lấy giá trị của các giới hạn hệ thống tại runtime.

```c
#include <unistd.h>
long sysconf(int name);
                                         Returns value of limit specified by name,
                              or –1 if limit is indeterminate or an error occurred
```

Đối số `name` là một trong các hằng số `_SC_*` được định nghĩa trong `<unistd.h>`, một số được liệt kê trong Bảng 11-1. Giá trị của giới hạn được trả về làm kết quả hàm.

Nếu giới hạn không thể xác định, `sysconf()` trả về `-1`. Nó cũng có thể trả về `-1` nếu xảy ra lỗi. (Lỗi duy nhất được quy định là `EINVAL`, chỉ ra rằng `name` không hợp lệ.) Để phân biệt trường hợp giới hạn không xác định với lỗi, ta phải đặt `errno` về 0 trước lời gọi; nếu lời gọi trả về `-1` và `errno` được đặt sau lời gọi thì đã xảy ra lỗi.

> Các giá trị giới hạn trả về bởi `sysconf()` (cũng như `pathconf()` và `fpathconf()`) luôn là số nguyên `(long)`. Trong phần lý luận của `sysconf()`, SUSv3 ghi nhận rằng chuỗi đã được xem xét như giá trị trả về có thể, nhưng bị loại vì sự phức tạp của việc triển khai và sử dụng.

Listing 11-1 minh họa cách dùng `sysconf()` để hiển thị các giới hạn hệ thống khác nhau. Chạy chương trình này trên một hệ thống Linux 2.6.31/x86-32 cho kết quả:

```
$ ./t_sysconf
   _SC_ARG_MAX: 2097152
   _SC_LOGIN_NAME_MAX: 256
   _SC_OPEN_MAX: 1024
   _SC_NGROUPS_MAX: 65536
   _SC_PAGESIZE: 4096
   _SC_RTSIG_MAX: 32
```

**Listing 11-1:** Sử dụng `sysconf()`

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– syslim/t_sysconf.c
#include "tlpi_hdr.h"
static void /* Print 'msg' plus sysconf() value for 'name' */
sysconfPrint(const char *msg, int name)
{
    long lim;
    errno = 0;
    lim = sysconf(name);
    if (lim != -1) {          /* Call succeeded, limit determinate */
        printf("%s %ld\n", msg, lim);
    } else {
        if (errno == 0)       /* Call succeeded, limit indeterminate */
            printf("%s (indeterminate)\n", msg);
        else                  /* Call failed */
            errExit("sysconf %s", msg);
    }
}
int
main(int argc, char *argv[])
{
    sysconfPrint("_SC_ARG_MAX: ",        _SC_ARG_MAX);
    sysconfPrint("_SC_LOGIN_NAME_MAX: ", _SC_LOGIN_NAME_MAX);
    sysconfPrint("_SC_OPEN_MAX: ",       _SC_OPEN_MAX);
    sysconfPrint("_SC_NGROUPS_MAX: ",    _SC_NGROUPS_MAX);
    sysconfPrint("_SC_PAGESIZE: ",       _SC_PAGESIZE);
    sysconfPrint("_SC_RTSIG_MAX: ",      _SC_RTSIG_MAX);
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– syslim/t_sysconf.c
```

SUSv3 yêu cầu giá trị trả về bởi `sysconf()` cho một giới hạn cụ thể phải là hằng số trong suốt vòng đời của process gọi. Ví dụ, ta có thể giả định giá trị trả về cho `_SC_PAGESIZE` sẽ không thay đổi trong khi process đang chạy.

> Trên Linux, có một số ngoại lệ (hợp lý) cho nhận định rằng giá trị giới hạn là hằng số trong suốt vòng đời process. Một process có thể dùng `setrlimit()` (Mục 36.2) để thay đổi các resource limit của process ảnh hưởng đến giá trị giới hạn được báo cáo bởi `sysconf()`: `RLIMIT_NOFILE`, xác định số file process có thể mở (`_SC_OPEN_MAX`); `RLIMIT_NPROC` (một resource limit thực ra không được quy định trong SUSv3), là giới hạn per-user về số process có thể tạo bởi process này (`_SC_CHILD_MAX`); và `RLIMIT_STACK`, từ Linux 2.6.23, xác định giới hạn về không gian cho phép dành cho command-line arguments và môi trường của process (`_SC_ARG_MAX`; xem manual page `execve(2)` để biết chi tiết).

---

## 11.3 Lấy Giới Hạn Liên Quan Đến File (và Tùy Chọn) tại Runtime

Các hàm `pathconf()` và `fpathconf()` cho phép ứng dụng lấy giá trị của các giới hạn liên quan đến file tại runtime.

```c
#include <unistd.h>
long pathconf(const char *pathname, int name);
long fpathconf(int fd, int name);
                                    Both return value of limit specified by name,
                              or –1 if limit is indeterminate or an error occurred
```

Điểm khác biệt duy nhất giữa `pathconf()` và `fpathconf()` là cách chỉ định file hay directory. Với `pathconf()`, chỉ định bằng pathname; với `fpathconf()`, chỉ định qua file descriptor (đã mở trước đó).

Đối số `name` là một trong các hằng số `_PC_*` được định nghĩa trong `<unistd.h>`, một số được liệt kê trong Bảng 11-1. Bảng 11-2 cung cấp thêm chi tiết về các hằng số `_PC_*` đã hiển thị trong Bảng 11-1.

Giá trị của giới hạn được trả về làm kết quả hàm. Ta có thể phân biệt giữa trả về không xác định và trả về lỗi theo cách tương tự như với `sysconf()`.

Khác với `sysconf()`, SUSv3 không yêu cầu các giá trị trả về bởi `pathconf()` và `fpathconf()` phải không đổi trong suốt vòng đời của process, vì ví dụ một file system có thể được unmount và remount với các đặc tính khác trong khi process đang chạy.

**Bảng 11-2:** Chi tiết về một số tên `_PC_*` của `pathconf()`

| Hằng số | Ghi chú |
|---------|---------|
| `_PC_NAME_MAX` | Với directory, trả về giá trị cho các file trong directory đó. Hành vi với các loại file khác không được quy định. |
| `_PC_PATH_MAX` | Với directory, trả về độ dài tối đa cho pathname tương đối từ directory này. Hành vi với các loại file khác không được quy định. |
| `_PC_PIPE_BUF` | Với FIFO hoặc pipe, trả về giá trị áp dụng cho file được tham chiếu. Với directory, giá trị áp dụng cho FIFO được tạo trong directory đó. Hành vi với các loại file khác không được quy định. |

Listing 11-2 cho thấy cách dùng `fpathconf()` để lấy các giới hạn khác nhau cho file được tham chiếu bởi standard input. Khi chạy chương trình này với standard input là một directory trên file system ext2:

```
$ ./t_fpathconf < .
   _PC_NAME_MAX: 255
   _PC_PATH_MAX: 4096
   _PC_PIPE_BUF: 4096
```

**Listing 11-2:** Sử dụng `fpathconf()`

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––– syslim/t_fpathconf.c
#include "tlpi_hdr.h"
static void /* Print 'msg' plus value of fpathconf(fd, name) */
fpathconfPrint(const char *msg, int fd, int name)
{
    long lim;
    errno = 0;
    lim = fpathconf(fd, name);
    if (lim != -1) {          /* Call succeeded, limit determinate */
        printf("%s %ld\n", msg, lim);
    } else {
        if (errno == 0)       /* Call succeeded, limit indeterminate */
            printf("%s (indeterminate)\n", msg);
        else                  /* Call failed */
            errExit("fpathconf %s", msg);
    }
}
int
main(int argc, char *argv[])
{
    fpathconfPrint("_PC_NAME_MAX: ", STDIN_FILENO, _PC_NAME_MAX);
    fpathconfPrint("_PC_PATH_MAX: ", STDIN_FILENO, _PC_PATH_MAX);
    fpathconfPrint("_PC_PIPE_BUF: ", STDIN_FILENO, _PC_PIPE_BUF);
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– syslim/t_fpathconf.c
```

---

## 11.4 Giới Hạn Không Xác Định

Đôi khi ta có thể thấy một số giới hạn hệ thống không được định nghĩa bởi hằng số giới hạn triển khai (ví dụ: `PATH_MAX`), và `sysconf()` hoặc `pathconf()` thông báo rằng giới hạn đó (ví dụ: `_PC_PATH_MAX`) là không xác định. Trong trường hợp này, ta có thể dùng một trong các chiến lược sau:

- Khi viết ứng dụng portable trên nhiều triển khai UNIX, ta có thể chọn dùng giá trị giới hạn tối thiểu do SUSv3 quy định. Đây là các hằng số có tên dạng `_POSIX_*_MAX`, mô tả trong Mục 11.1. Đôi khi cách này không khả thi vì giới hạn quá thấp, như trường hợp của `_POSIX_PATH_MAX` và `_POSIX_OPEN_MAX`.

- Trong một số trường hợp, giải pháp thực tế là bỏ qua việc kiểm tra giới hạn, và thay vào đó thực hiện lời gọi system call hoặc library function liên quan. (Lập luận tương tự cũng áp dụng với một số tùy chọn SUSv3 mô tả trong Mục 11.5.) Nếu lời gọi thất bại và `errno` chỉ ra rằng lỗi xảy ra do vượt quá giới hạn hệ thống, ta có thể thử lại, điều chỉnh hành vi ứng dụng khi cần. Ví dụ, hầu hết các triển khai UNIX áp đặt giới hạn về số realtime signal có thể xếp hàng cho một process. Khi đạt giới hạn, các nỗ lực gửi thêm signal (dùng `sigqueue()`) thất bại với lỗi `EAGAIN`. Trong trường hợp này, process gửi có thể đơn giản thử lại, có thể sau một khoảng thời gian chờ. Tương tự, cố gắng mở file có tên quá dài trả về lỗi `ENAMETOOLONG`, và ứng dụng có thể xử lý bằng cách thử lại với tên ngắn hơn.

- Ta có thể tự viết chương trình hoặc hàm để suy luận hoặc ước lượng giới hạn. Trong mỗi trường hợp, lời gọi `sysconf()` hoặc `pathconf()` liên quan được thực hiện, và nếu giới hạn không xác định, hàm trả về giá trị "ước đoán tốt". Dù không hoàn hảo, giải pháp như vậy thường khả thi trong thực tế.

- Ta có thể dùng công cụ như GNU Autoconf, một công cụ có thể mở rộng để xác định sự tồn tại và cài đặt của các tính năng và giới hạn hệ thống khác nhau. Chương trình Autoconf tạo ra các header file dựa trên thông tin nó xác định, và các file này có thể được include trong các chương trình C.

---

## 11.5 Tùy Chọn Hệ Thống

Ngoài việc quy định giới hạn cho các tài nguyên hệ thống khác nhau, SUSv3 còn quy định các tùy chọn mà một triển khai UNIX có thể hỗ trợ. Chúng bao gồm hỗ trợ các tính năng như realtime signal, POSIX shared memory, job control, và POSIX thread. Với một vài ngoại lệ, các triển khai không bắt buộc phải hỗ trợ các tùy chọn này. Thay vào đó, SUSv3 cho phép một triển khai thông báo — cả tại compile time lẫn runtime — liệu nó có hỗ trợ một tính năng cụ thể không.

Một triển khai có thể quảng cáo hỗ trợ một tùy chọn SUSv3 tại compile time bằng cách định nghĩa một hằng số tương ứng trong `<unistd.h>`. Mỗi hằng số như vậy bắt đầu bằng tiền tố chỉ ra chuẩn từ đó nó xuất phát (ví dụ: `_POSIX_` hoặc `_XOPEN_`).

Mỗi hằng số tùy chọn, nếu được định nghĩa, có một trong các giá trị sau:

- Giá trị `-1` có nghĩa tùy chọn không được hỗ trợ. Trong trường hợp này, các header file, kiểu dữ liệu và giao diện hàm liên quan đến tùy chọn không cần được định nghĩa bởi triển khai. Ta có thể cần xử lý khả năng này bằng biên dịch có điều kiện dùng chỉ thị tiền xử lý `#if`.
- Giá trị `0` có nghĩa tùy chọn có thể được hỗ trợ. Ứng dụng phải kiểm tra tại runtime xem tùy chọn có được hỗ trợ không.
- Giá trị lớn hơn `0` có nghĩa tùy chọn được hỗ trợ. Tất cả header file, kiểu dữ liệu và giao diện hàm liên quan đến tùy chọn này được định nghĩa và hoạt động theo quy định. Trong nhiều trường hợp, SUSv3 yêu cầu giá trị dương này phải là `200112L`, một hằng số tương ứng với năm và tháng SUSv3 được phê duyệt là chuẩn. (Giá trị tương tự cho SUSv4 là `200809L`.)

Khi hằng số được định nghĩa với giá trị `0`, ứng dụng có thể dùng hàm `sysconf()` và `pathconf()` (hoặc `fpathconf()`) để kiểm tra tại runtime xem tùy chọn có được hỗ trợ không. Các đối số `name` được truyền cho các hàm này thường có cùng dạng với các hằng số compile-time tương ứng, nhưng với tiền tố được thay thế bằng `_SC_` hoặc `_PC_`. Triển khai phải cung cấp ít nhất các header file, hằng số, và giao diện hàm cần thiết để thực hiện kiểm tra runtime.

> SUSv3 không rõ ràng về việc hằng số tùy chọn không được định nghĩa có cùng ý nghĩa với định nghĩa hằng số với giá trị `0` ("tùy chọn có thể được hỗ trợ") hay với giá trị `-1` ("tùy chọn không được hỗ trợ"). Ủy ban chuẩn sau đó đã giải quyết rằng trường hợp này có nghĩa giống như định nghĩa hằng số với giá trị `-1`, và SUSv4 nêu rõ điều này.

Bảng 11-3 liệt kê một số tùy chọn được quy định trong SUSv3. Cột đầu tiên của bảng cho tên hằng số compile-time liên quan đến tùy chọn (định nghĩa trong `<unistd.h>`), cũng như đối số tên `sysconf()` (`_SC_*`) hoặc `pathconf()` (`_PC_*`) tương ứng.

**Bảng 11-3:** Một số tùy chọn SUSv3

| Tên tùy chọn (hằng số) / (tên `sysconf()` / `pathconf()`) | Mô tả | Ghi chú |
|-------------------------------------------------------------|-------|---------|
| `_POSIX_ASYNCHRONOUS_IO` / `(_SC_ASYNCHRONOUS_IO)` | Asynchronous I/O | |
| `_POSIX_CHOWN_RESTRICTED` / `(_PC_CHOWN_RESTRICTED)` | Chỉ process có đặc quyền mới có thể dùng `chown()` và `fchown()` để thay đổi user ID và group ID của file sang giá trị tùy ý (Mục 15.3.2) | * |
| `_POSIX_JOB_CONTROL` / `(_SC_JOB_CONTROL)` | Job Control (Mục 34.7) | + |
| `_POSIX_MESSAGE_PASSING` / `(_SC_MESSAGE_PASSING)` | POSIX Message Queue (Chương 52) | |
| `_POSIX_PRIORITY_SCHEDULING` / `(_SC_PRIORITY_SCHEDULING)` | Process Scheduling (Mục 35.3) | |
| `_POSIX_REALTIME_SIGNALS` / `(_SC_REALTIME_SIGNALS)` | Realtime Signals Extension (Mục 22.8) | |
| `_POSIX_SAVED_IDS` / (none) | Process có saved set-user-ID và saved set-group-ID (Mục 9.4) | + |
| `_POSIX_SEMAPHORES` / `(_SC_SEMAPHORES)` | POSIX Semaphore (Chương 53) | |
| `_POSIX_SHARED_MEMORY_OBJECTS` / `(_SC_SHARED_MEMORY_OBJECTS)` | POSIX Shared Memory Object (Chương 54) | |
| `_POSIX_THREADS` / `(_SC_THREADS)` | POSIX Thread | |
| `_XOPEN_UNIX` / `(_SC_XOPEN_UNIX)` | XSI extension được hỗ trợ (Mục 1.3.4) | |

Ghi chú: `+` = tùy chọn bắt buộc theo SUSv3; `*` = hằng số compile-time phải có giá trị khác `-1`.

---

## 11.6 Tóm Tắt

SUSv3 quy định các giới hạn mà một triển khai có thể áp đặt và các tùy chọn hệ thống mà một triển khai có thể hỗ trợ.

Thường không nên hard-code các giả định về giới hạn và tùy chọn hệ thống vào chương trình, vì chúng có thể thay đổi giữa các triển khai cũng như trên một triển khai đơn lẻ, cả tại runtime lẫn giữa các file system. Do đó, SUSv3 quy định các phương pháp mà triển khai có thể thông báo các giới hạn và tùy chọn nó hỗ trợ. Đối với hầu hết các giới hạn, SUSv3 quy định giá trị tối thiểu mà tất cả các triển khai phải hỗ trợ. Ngoài ra, mỗi triển khai có thể thông báo các giới hạn và tùy chọn cụ thể của mình tại compile time (qua định nghĩa hằng số trong `<limits.h>` hoặc `<unistd.h>`) và/hoặc runtime (qua lời gọi `sysconf()`, `pathconf()`, hoặc `fpathconf()`). Các kỹ thuật này cũng có thể được dùng để tìm hiểu các tùy chọn SUSv3 mà một triển khai hỗ trợ. Trong một số trường hợp, có thể không thể xác định một giới hạn cụ thể bằng một trong hai phương pháp này. Đối với các giới hạn không xác định như vậy, chúng ta phải dùng các kỹ thuật đặc biệt để xác định giới hạn mà ứng dụng nên tuân theo.

---

## 11.7 Bài Tập

- **11-1.** Thử chạy chương trình trong Listing 11-1 trên các triển khai UNIX khác nếu bạn có quyền truy cập chúng.
- **11-2.** Thử chạy chương trình trong Listing 11-2 trên các file system khác nhau.
