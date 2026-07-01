## Chương 10
# Thời Gian (Time)

Trong chương trình, chúng ta có thể quan tâm đến hai loại thời gian:

- **Real time**: Là thời gian được đo từ một điểm chuẩn nào đó (calendar time) hoặc từ một điểm cố định (thường là điểm bắt đầu) trong vòng đời của process (elapsed time hay wall clock time). Lấy calendar time hữu ích cho các chương trình đánh timestamp cho bản ghi database hoặc file.
- **Process time**: Là lượng CPU time được process sử dụng. Đo process time hữu ích để kiểm tra hoặc tối ưu hiệu năng chương trình.

Hầu hết kiến trúc máy tính có hardware clock tích hợp cho phép kernel đo real time và process time.

---

## 10.1 Calendar Time

Bất kể vị trí địa lý, hệ thống UNIX biểu diễn thời gian nội bộ như số giây kể từ **Epoch** — tức là kể từ nửa đêm sáng ngày 1 tháng 1 năm 1970, UTC (Coordinated Universal Time, trước đây được biết là GMT). Calendar time được lưu trong biến kiểu `time_t`, một kiểu integer được quy định bởi SUSv3.

> Trên hệ thống Linux 32-bit, `time_t` là signed integer có thể biểu diễn ngày trong phạm vi 13 tháng 12 năm 1901 đến 19 tháng 1 năm 2038. Nhiều hệ thống UNIX 32-bit hiện tại đối mặt với vấn đề Year 2038 về mặt lý thuyết.

System call `gettimeofday()` trả về calendar time trong buffer được trỏ bởi `tv`.

```c
#include <sys/time.h>
int gettimeofday(struct timeval *tv, struct timezone *tz);
                                             Returns 0 on success, or –1 on error
```

Đối số `tv` là pointer đến structure có dạng:

```c
struct timeval {
    time_t      tv_sec;   /* Seconds since 00:00:00, 1 Jan 1970 UTC */
    suseconds_t tv_usec;  /* Additional microseconds (long int) */
};
```

Đối số `tz` là artifact lịch sử và luôn phải chỉ định là `NULL`.

System call `time()` trả về số giây kể từ Epoch.

```c
#include <time.h>
time_t time(time_t *timep);
             Returns number of seconds since the Epoch, or (time_t) –1 on error
```

Thường dùng đơn giản như: `t = time(NULL);`

---

## 10.2 Hàm Chuyển Đổi Thời Gian

Hình 10-1 cho thấy các hàm dùng để chuyển đổi giữa giá trị `time_t` và các định dạng thời gian khác, bao gồm biểu diễn có thể in. Các hàm này bảo vệ ta khỏi sự phức tạp của timezone, DST, và vấn đề bản địa hóa.

```
Kernel ──time()──► time_t ──gmtime()*──► struct tm ──asctime()──► chuỗi cố định
       ──gettimeofday()──► timeval      ──localtime()*──►           ctime()*──►
                                        ──mktime()*──►   time_t
                                        ──strftime()*+──► chuỗi bản địa hóa
                                    strptime()*+ ◄──
```

### 10.2.1 Chuyển Đổi time_t thành Dạng In Được

Hàm `ctime()` cung cấp phương pháp đơn giản chuyển đổi `time_t` thành dạng có thể in.

```c
#include <time.h>
char *ctime(const time_t *timep);
                         Returns pointer to statically allocated string terminated
                                   by newline and \0 on success, or NULL on error
```

Trả về chuỗi 26 byte, ví dụ: `Wed Jun  8 14:22:34 2011\n\0`. Hàm tự động tính đến timezone và cài đặt DST. Chuỗi trả về được cấp phát tĩnh, lời gọi tiếp theo sẽ ghi đè lên.

Phiên bản reentrant: `ctime_r()`.

### 10.2.2 Chuyển Đổi Giữa time_t và Broken-Down Time

Các hàm `gmtime()` và `localtime()` chuyển đổi `time_t` thành **broken-down time** — thời gian phân rã thành các thành phần riêng lẻ.

```c
#include <time.h>
struct tm *gmtime(const time_t *timep);
struct tm *localtime(const time_t *timep);
                      Both return a pointer to a statically allocated broken-down
                                       time structure on success, or NULL on error
```

`gmtime()` chuyển đổi sang UTC. `localtime()` tính đến timezone và DST để trả về local time. Phiên bản reentrant: `gmtime_r()` và `localtime_r()`.

Structure `tm` có dạng:

```c
struct tm {
    int tm_sec;    /* Seconds (0-60) */
    int tm_min;    /* Minutes (0-59) */
    int tm_hour;   /* Hours (0-23) */
    int tm_mday;   /* Day of the month (1-31) */
    int tm_mon;    /* Month (0-11) */
    int tm_year;   /* Year since 1900 */
    int tm_wday;   /* Day of the week (Sunday = 0) */
    int tm_yday;   /* Day in the year (0-365; 1 Jan = 0) */
    int tm_isdst;  /* Daylight saving time flag
                      > 0: DST is in effect;
                      = 0: DST is not in effect;
                      < 0: DST information not available */
};
```

Hàm `mktime()` dịch broken-down time (biểu diễn local time) thành `time_t`.

```c
#include <time.h>
time_t mktime(struct tm *timeptr);
                      Returns seconds since the Epoch corresponding to timeptr
                                               on success, or (time_t) –1 on error
```

`mktime()` có thể sửa đổi structure được trỏ bởi `timeptr` — đảm bảo `tm_wday` và `tm_yday` được đặt đúng. Các trường có giá trị ngoài phạm vi sẽ được điều chỉnh (ví dụ: `tm_sec = 123` trở thành `tm_sec = 3, tm_min += 2`). Điều này cho phép thực hiện phép tính ngày và giờ.

### 10.2.3 Chuyển Đổi Giữa Broken-Down Time và Dạng In Được

**Từ broken-down time sang dạng có thể in:**

Hàm `asctime()` trả về pointer đến chuỗi được cấp phát tĩnh cùng định dạng như `ctime()`.

```c
#include <time.h>
char *asctime(const struct tm *timeptr);
                      Returns pointer to statically allocated string terminated by
                                      newline and \0 on success, or NULL on error
```

Phiên bản reentrant: `asctime_r()`.

Hàm `strftime()` cung cấp kiểm soát chính xác hơn khi chuyển đổi broken-down time thành dạng có thể in.

```c
#include <time.h>
size_t strftime(char *outstr, size_t maxsize, const char *format,
                const struct tm *timeptr);
                           Returns number of bytes placed in outstr (excluding
                               terminating null byte) on success, or 0 on error
```

Đối số `format` chứa chuỗi với các conversion specification bắt đầu bằng `%`. Khác với `ctime()` và `asctime()`, `strftime()` không thêm newline ở cuối.

**Bảng 10-1:** Một số conversion specifier của `strftime()`

| Specifier | Mô tả | Ví dụ |
|-----------|-------|-------|
| `%%` | Ký tự % | `%` |
| `%a` | Tên ngày trong tuần viết tắt | `Tue` |
| `%A` | Tên ngày trong tuần đầy đủ | `Tuesday` |
| `%b`, `%h` | Tên tháng viết tắt | `Feb` |
| `%B` | Tên tháng đầy đủ | `February` |
| `%c` | Ngày và giờ | `Tue Feb  1 21:39:46 2011` |
| `%d` | Ngày trong tháng (2 chữ số) | `01` |
| `%F` | Ngày ISO (như `%Y-%m-%d`) | `2011-02-01` |
| `%H` | Giờ (đồng hồ 24 giờ) | `21` |
| `%M` | Phút | `39` |
| `%S` | Giây (00 đến 60) | `46` |
| `%T` | Thời gian (như `%H:%M:%S`) | `21:39:46` |
| `%Y` | Năm 4 chữ số | `2011` |
| `%Z` | Tên timezone | `CET` |

Hàm `currTime()` được cung cấp để lấy thời gian hiện tại dưới dạng chuỗi:

```c
#include "curr_time.h"
char *currTime(const char *format);
                    Returns pointer to statically allocated string, or NULL on error
```

**Từ dạng có thể in sang broken-down time:**

Hàm `strptime()` là hàm đảo của `strftime()`, chuyển đổi chuỗi ngày giờ thành broken-down time.

```c
#define _XOPEN_SOURCE
#include <time.h>
char *strptime(const char *str, const char *format, struct tm *timeptr);
                               Returns pointer to next unprocessed character in
                                                   str on success, or NULL on error
```

Khi thành công, `strptime()` trả về pointer đến ký tự tiếp theo chưa xử lý trong `str`. `strptime()` không bao giờ đặt trường `tm_isdst` của structure `tm`.

---

## 10.3 Timezone

Các quốc gia (và đôi khi vùng trong một quốc gia) sử dụng timezone và chế độ DST khác nhau. Thông tin timezone được lưu trong `/usr/share/zoneinfo`. Timezone cục bộ của hệ thống được định nghĩa bởi `/etc/localtime`, thường là symlink đến một file trong `/usr/share/zoneinfo`.

**Chỉ định timezone cho chương trình:**

Để chỉ định timezone khi chạy chương trình, đặt biến môi trường `TZ` thành chuỗi bao gồm dấu hai chấm (`:`) theo sau bởi tên timezone (ví dụ: `TZ=":Pacific/Auckland"`). Cài đặt timezone tự động ảnh hưởng đến `ctime()`, `localtime()`, `mktime()`, và `strftime()`.

Mỗi hàm này dùng `tzset(3)` để khởi tạo ba biến toàn cục:

```c
char *tzname[2]; /* Tên timezone và alternate (DST) timezone */
int daylight;    /* Khác 0 nếu có alternate (DST) timezone */
long timezone;   /* Chênh lệch giây giữa UTC và local standard time */
```

---

## 10.4 Locale

Locale là "tập con của môi trường người dùng phụ thuộc vào ngôn ngữ và quy ước văn hóa." Thông tin locale được duy trì trong cây thư mục dưới `/usr/share/locale`. Mỗi subdirectory chứa thông tin về locale cụ thể với tên theo quy ước: `language[_territory[.codeset]][@modifier]`.

**Bảng 10-2:** Nội dung của locale-specific subdirectory

| Tên file | Mục đích |
|----------|---------|
| `LC_CTYPE` | Phân loại ký tự và quy tắc chuyển đổi chữ hoa/thường |
| `LC_COLLATE` | Quy tắc collation (sắp xếp ký tự) |
| `LC_MONETARY` | Quy tắc định dạng giá trị tiền tệ |
| `LC_NUMERIC` | Quy tắc định dạng số không phải tiền tệ |
| `LC_TIME` | Quy tắc định dạng ngày và giờ |
| `LC_MESSAGES` | File chỉ định định dạng và giá trị cho câu trả lời yes/no |

**Chỉ định locale cho chương trình:**

Hàm `setlocale()` dùng để đặt và truy vấn locale hiện tại của chương trình.

```c
#include <locale.h>
char *setlocale(int category, const char *locale);
              Returns pointer to a (usually statically allocated) string identifying
                            the new or current locale on success, or NULL on error
```

Đối số `category` chọn phần locale nào cần đặt hoặc truy vấn (ví dụ: `LC_ALL`, `LC_TIME`). Chỉ định `locale` là chuỗi rỗng `""` để lấy cài đặt locale từ biến môi trường:

```c
setlocale(LC_ALL, "");
```

Phải thực hiện lời gọi này để chương trình nhận biết các biến môi trường locale.

---

## 10.5 Cập Nhật System Clock

Hai giao diện cập nhật system clock: `settimeofday()` và `adjtime()`. Chúng yêu cầu caller có đặc quyền (`CAP_SYS_TIME`).

`settimeofday()` đặt calendar time của hệ thống theo số giây và microsecond trong `timeval`:

```c
#define _BSD_SOURCE
#include <sys/time.h>
int settimeofday(const struct timeval *tv, const struct timezone *tz);
                                             Returns 0 on success, or –1 on error
```

Thay đổi đột ngột có thể ảnh hưởng tiêu cực đến các ứng dụng phụ thuộc vào monotonically increasing clock (như `make`, database với timestamp). Để thay đổi nhỏ, tốt hơn dùng `adjtime()`, điều chỉnh dần dần.

```c
#define _BSD_SOURCE
#include <sys/time.h>
int adjtime(struct timeval *delta, struct timeval *olddelta);
                                             Returns 0 on success, or –1 on error
```

Đối số `delta` chỉ định số giây và microsecond cần thay đổi thời gian. Nếu dương, thêm một lượng nhỏ mỗi giây; nếu âm, làm chậm clock. `olddelta` trả về lượng điều chỉnh chưa hoàn thành (có thể `NULL`).

---

## 10.6 Software Clock (Jiffies)

Độ chính xác của các time-related system call bị giới hạn bởi độ phân giải của software clock hệ thống, đo thời gian theo đơn vị **jiffy**. Kích thước jiffy được định nghĩa bởi hằng số `HZ` trong kernel source code.

Trên Linux/x86-32 trong kernel 2.4, software clock chạy ở 100 Hz (jiffy = 10 ms). Trong kernel 2.6.0, nâng lên 1000 Hz. Từ kernel 2.6.13, tốc độ là tùy chọn cấu hình kernel (100, 250, hoặc 1000 Hz, mặc định 250). Từ kernel 2.6.20, thêm 300 Hz.

---

## 10.7 Process Time

Process time là lượng CPU time được process sử dụng kể từ khi tạo. Kernel chia CPU time thành hai thành phần:

- **User CPU time**: Thời gian thực thi trong user mode.
- **System CPU time**: Thời gian thực thi trong kernel mode (system call, xử lý page fault, v.v.).

System call `times()` lấy thông tin process time, trả về trong structure được trỏ bởi `buf`.

```c
#include <sys/times.h>
clock_t times(struct tms *buf);
                   Returns number of clock ticks (sysconf(_SC_CLK_TCK)) since
                      "arbitrary" time in past on success, or (clock_t) –1 on error
```

```c
struct tms {
    clock_t tms_utime;  /* User CPU time used by caller */
    clock_t tms_stime;  /* System CPU time used by caller */
    clock_t tms_cutime; /* User CPU time of all (waited for) children */
    clock_t tms_cstime; /* System CPU time of all (waited for) children */
};
```

Kiểu `clock_t` đo thời gian theo đơn vị clock tick. Gọi `sysconf(_SC_CLK_TCK)` để lấy số clock tick mỗi giây, rồi chia giá trị `clock_t` để chuyển sang giây.

Giá trị trả về của `times()` là elapsed time tính từ điểm tùy ý trong quá khứ. Cách đáng tin cậy để đo elapsed time là dùng `gettimeofday()`.

Hàm `clock()` cung cấp giao diện đơn giản hơn để lấy tổng CPU time (user + system):

```c
#include <time.h>
clock_t clock(void);
                    Returns total CPU time used by calling process measured in
                                           CLOCKS_PER_SEC, or (clock_t) –1 on error
```

`CLOCKS_PER_SEC` được POSIX.1 cố định ở 1 triệu. Phải chia giá trị trả về cho `CLOCKS_PER_SEC` để có số giây.

---

## 10.8 Tóm Tắt

Real time tương ứng với định nghĩa thời gian thông thường. Calendar time đo từ điểm chuẩn (Epoch); elapsed time đo từ điểm bắt đầu của process.

Process time là lượng CPU time được process sử dụng, chia thành thành phần user và system.

Nhiều system call cho phép lấy và đặt giá trị system clock, và nhiều hàm thư viện cho phép chuyển đổi giữa calendar time và các định dạng khác, gồm broken-down time và chuỗi có thể đọc được.

---

## 10.9 Bài Tập

**10-1.** Giả sử `sysconf(_SC_CLK_TCK)` trả về 100. Nếu `clock_t` là unsigned 32-bit integer, bao lâu thì giá trị trả về bởi `times()` sẽ cycle về 0? Tính tương tự cho `CLOCKS_PER_SEC` được trả về bởi `clock()`.
