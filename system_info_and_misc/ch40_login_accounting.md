## Chương 40
# Ghi Nhận Thông Tin Đăng Nhập (Login Accounting)

Login accounting (ghi nhận thông tin đăng nhập) liên quan đến việc ghi lại người dùng nào hiện đang đăng nhập vào hệ thống, cũng như ghi lại lịch sử đăng nhập và đăng xuất trong quá khứ. Chương này xem xét các file login accounting và các hàm thư viện được dùng để lấy và cập nhật thông tin chứa trong chúng. Chúng ta cũng mô tả các bước mà ứng dụng cung cấp dịch vụ đăng nhập cần thực hiện để cập nhật các file này khi người dùng đăng nhập và đăng xuất.

---

## 40.1 Tổng Quan về Các File utmp và wtmp

Các hệ thống UNIX duy trì hai file dữ liệu chứa thông tin về người dùng đăng nhập và đăng xuất khỏi hệ thống:

- **File utmp** lưu một bản ghi về người dùng hiện đang đăng nhập vào hệ thống (cũng như một số thông tin khác sẽ mô tả sau). Khi mỗi người dùng đăng nhập, một bản ghi được ghi vào file utmp. Một trường trong bản ghi này, `ut_user`, lưu tên đăng nhập của người dùng. Bản ghi này sau đó bị xóa khi đăng xuất. Các chương trình như `who(1)` dùng thông tin trong file utmp để hiển thị danh sách người dùng hiện đang đăng nhập.

- **File wtmp** là audit trail (vết kiểm tra) của tất cả các lần đăng nhập và đăng xuất của người dùng (cũng như một số thông tin khác sẽ mô tả sau). Mỗi lần đăng nhập, một bản ghi chứa thông tin giống như ghi vào file utmp được nối vào file wtmp. Khi đăng xuất, một bản ghi khác được nối vào file. Bản ghi này chứa cùng thông tin, ngoại trừ trường `ut_user` bị đặt về không. Lệnh `last(1)` có thể dùng để hiển thị và lọc nội dung file wtmp.

Trên Linux, file utmp nằm tại `/var/run/utmp` và file wtmp nằm tại `/var/log/wtmp`. Nói chung, các ứng dụng không cần biết các pathname này vì chúng được biên dịch vào glibc. Các chương trình cần tham chiếu đến vị trí của các file này nên dùng các hằng số pathname `_PATH_UTMP` và `_PATH_WTMP`, được định nghĩa trong `<paths.h>` (và `<utmpx.h>`), thay vì hard-code pathname vào chương trình.

> SUSv3 không chuẩn hóa bất kỳ tên symbolic nào cho pathname của các file utmp và wtmp. Tên `_PATH_UTMP` và `_PATH_WTMP` được dùng trên Linux và BSD. Nhiều triển khai UNIX khác thay vào đó định nghĩa các hằng số `UTMP_FILE` và `WTMP_FILE` cho các pathname này. Linux cũng định nghĩa các tên này trong `<utmp.h>`, nhưng không định nghĩa chúng trong `<utmpx.h>` hay `<paths.h>`.

---

## 40.2 utmpx API

Các file utmp và wtmp đã có trong hệ thống UNIX từ thời kỳ đầu, nhưng đã trải qua quá trình phát triển và phân kỳ liên tục giữa các triển khai UNIX khác nhau, đặc biệt là BSD so với System V. System V Release 4 đã mở rộng đáng kể API, trong quá trình đó tạo ra structure `utmpx` mới (song song) và các file `utmpx` và `wtmpx` liên quan. Chữ `x` cũng được đưa vào tên các header file và các hàm bổ sung để xử lý các file mới này. Nhiều triển khai UNIX khác cũng thêm các mở rộng của riêng họ vào API.

Trong chương này, chúng ta mô tả Linux utmpx API, là sự kết hợp của các triển khai BSD và System V. Linux không theo System V trong việc tạo các file `utmpx` và `wtmpx` song song; thay vào đó, các file utmp và wtmp chứa tất cả thông tin cần thiết. Tuy nhiên, để tương thích với các triển khai UNIX khác, Linux cung cấp cả API utmp truyền thống lẫn API utmpx dẫn xuất từ System V để truy cập nội dung các file này. Trên Linux, hai API này trả về đúng cùng thông tin. (Một trong số ít khác biệt giữa hai API là utmp API chứa các phiên bản reentrant của một số hàm, còn utmpx API thì không.) Tuy nhiên, chúng ta giới hạn thảo luận ở giao diện utmpx, vì đó là API được quy định trong SUSv3 và do đó được ưa thích cho tính portable đến các triển khai UNIX khác.

Đặc tả SUSv3 không bao gồm tất cả các khía cạnh của utmpx API (ví dụ: vị trí của các file utmp và wtmp không được quy định). Nội dung chính xác của các file login accounting có một số khác biệt giữa các triển khai, và nhiều triển khai cung cấp thêm các hàm login accounting không được quy định trong SUSv3.

---

## 40.3 Structure utmpx

Các file utmp và wtmp bao gồm các bản ghi `utmpx`. Structure `utmpx` được định nghĩa trong `<utmpx.h>`, như được hiển thị trong Listing 40-1.

Đặc tả SUSv3 của structure `utmpx` không bao gồm các trường `ut_host`, `ut_exit`, `ut_session`, hay `ut_addr_v6`. Các trường `ut_host` và `ut_exit` có mặt trên hầu hết các triển khai khác; `ut_session` có mặt trên một số triển khai khác; và `ut_addr_v6` là đặc thù của Linux. SUSv3 quy định các trường `ut_line` và `ut_user`, nhưng để không xác định độ dài của chúng.

Kiểu dữ liệu `int32_t` dùng để định nghĩa trường `ut_addr_v6` của structure `utmpx` là số nguyên 32-bit.

**Listing 40-1:** Định nghĩa structure utmpx

```c
#define _GNU_SOURCE /* Without _GNU_SOURCE the two field
struct exit_status {    names below are prepended by "__" */
    short e_termination; /* Process termination status (signal) */
    short e_exit;        /* Process exit status */
};
#define __UT_LINESIZE 32
#define __UT_NAMESIZE 32
#define __UT_HOSTSIZE 256
struct utmpx {
    short   ut_type;               /* Type of record */
    pid_t   ut_pid;                /* PID of login process */
    char    ut_line[__UT_LINESIZE]; /* Terminal device name */
    char    ut_id[4];              /* Suffix from terminal name, or
                                      ID field from inittab(5) */
    char    ut_user[__UT_NAMESIZE]; /* Username */
    char    ut_host[__UT_HOSTSIZE]; /* Hostname for remote login, or kernel
                                      version for run-level messages */
    struct exit_status ut_exit;    /* Exit status of process marked
                                      as DEAD_PROCESS (not filled
                                      in by init(8) on Linux) */
    long    ut_session;            /* Session ID */
    struct timeval ut_tv;          /* Time when entry was made */
    int32_t ut_addr_v6[4];         /* IP address of remote host (IPv4
                                      address uses just ut_addr_v6[0],
                                      with other elements set to 0) */
    char    __unused[20];          /* Reserved for future use */
};
```

Mỗi trường chuỗi trong structure `utmpx` kết thúc bằng null trừ khi nó lấp đầy hoàn toàn mảng tương ứng.

Đối với login process, thông tin được lưu trong các trường `ut_line` và `ut_id` được lấy từ tên của thiết bị terminal. Trường `ut_line` chứa tên file đầy đủ của thiết bị terminal. Trường `ut_id` chứa phần hậu tố của tên file — tức là, chuỗi sau `tty`, `pts`, hoặc `pty` (hai cái sau là cho pseudoterminal kiểu System-V và BSD). Do đó, với terminal `/dev/tty2`, `ut_line` sẽ là `tty2` và `ut_id` sẽ là `2`.

Trong môi trường windowing, một số terminal emulator dùng trường `ut_session` để ghi session ID cho terminal window. (Xem Mục 34.3 để giải thích về session ID.)

Trường `ut_type` là số nguyên định nghĩa loại bản ghi được ghi vào file. Các hằng số sau (với giá trị số tương ứng trong ngoặc đơn) có thể được dùng làm giá trị cho trường này:

#### EMPTY (0)

Bản ghi này không chứa thông tin accounting hợp lệ.

#### RUN_LVL (1)

Bản ghi này chỉ ra sự thay đổi run-level của hệ thống trong quá trình khởi động hay tắt máy. (Thông tin về run-level có trong manual page `init(8)`.) Feature test macro `_GNU_SOURCE` phải được định nghĩa để lấy định nghĩa của hằng số này từ `<utmpx.h>`.

#### BOOT_TIME (2)

Bản ghi này chứa thời gian khởi động hệ thống trong trường `ut_tv`. Tác giả thông thường của các bản ghi `RUN_LVL` và `BOOT_TIME` là `init`. Các bản ghi này được ghi vào cả file utmp lẫn file wtmp.

#### NEW_TIME (3)

Bản ghi này chứa thời gian mới sau khi thay đổi system clock, được ghi trong trường `ut_tv`.

#### OLD_TIME (4)

Bản ghi này chứa thời gian cũ trước khi thay đổi system clock, được ghi trong trường `ut_tv`. Các bản ghi kiểu `OLD_TIME` và `NEW_TIME` được ghi vào file utmp và wtmp bởi NTP daemon (hoặc daemon tương tự) khi nó thực hiện các thay đổi với system clock.

#### INIT_PROCESS (5)

Đây là bản ghi cho một process được spawn bởi `init`, như process `getty`. Xem manual page `inittab(5)` để biết chi tiết.

#### LOGIN_PROCESS (6)

Đây là bản ghi cho một session leader process của lần đăng nhập người dùng, như process `login(1)`.

#### USER_PROCESS (7)

Đây là bản ghi cho một user process, thường là login session, với tên người dùng xuất hiện trong trường `ut_user`. Login session có thể được khởi động bởi `login(1)` hoặc bởi một ứng dụng cung cấp tính năng remote login như `ftp` hoặc `ssh`.

#### DEAD_PROCESS (8)

Bản ghi này xác định một process đã kết thúc.

Chúng ta hiển thị các giá trị số của các hằng số này vì nhiều ứng dụng phụ thuộc vào các hằng số có thứ tự số trên. Ví dụ, trong mã nguồn của chương trình `agetty`, ta thấy các kiểm tra như:

```c
utp->ut_type >= INIT_PROCESS && utp->ut_type <= DEAD_PROCESS
```

Các bản ghi kiểu `INIT_PROCESS` thường tương ứng với các lời gọi `getty(8)` (hoặc chương trình tương tự như `agetty(8)` hay `mingetty(8)`). Khi khởi động hệ thống, process `init` tạo một child cho mỗi terminal line và virtual console, và mỗi child exec chương trình `getty`. Chương trình `getty` mở terminal, nhắc người dùng nhập tên đăng nhập, và sau đó exec `login(1)`. Sau khi xác thực người dùng thành công và thực hiện nhiều bước khác, `login` fork một child để exec login shell của người dùng. Toàn bộ vòng đời của login session như vậy được biểu diễn bởi bốn bản ghi được ghi vào file wtmp theo thứ tự sau:

- Bản ghi `INIT_PROCESS`, được ghi bởi `init`;
- Bản ghi `LOGIN_PROCESS`, được ghi bởi `getty`;
- Bản ghi `USER_PROCESS`, được ghi bởi `login`; và
- Bản ghi `DEAD_PROCESS`, được ghi bởi `init` khi nó phát hiện cái chết của child login process (xảy ra khi người dùng đăng xuất).

> Một số phiên bản `init` spawn process `getty` trước khi cập nhật file wtmp. Do đó, `init` và `getty` chạy đua với nhau để cập nhật file wtmp, với kết quả là các bản ghi `INIT_PROCESS` và `LOGIN_PROCESS` có thể được ghi theo thứ tự ngược lại so với mô tả trong phần chính.

---

## 40.4 Lấy Thông Tin từ Các File utmp và wtmp

Các hàm mô tả trong mục này lấy bản ghi từ các file chứa bản ghi định dạng utmpx. Theo mặc định, các hàm này dùng file utmp chuẩn, nhưng có thể thay đổi bằng hàm `utmpxname()` (mô tả bên dưới).

Các hàm này dùng khái niệm vị trí hiện tại trong file mà chúng đang lấy bản ghi. Vị trí này được cập nhật bởi mỗi hàm.

Hàm `setutxent()` tua lại file utmp về đầu.

```c
#include <utmpx.h>
void setutxent(void);
```

Thông thường, ta nên gọi `setutxent()` trước khi dùng bất kỳ hàm `getutx*()` nào (mô tả bên dưới). Điều này ngăn ngừa sự nhầm lẫn có thể xảy ra nếu một hàm của bên thứ ba mà ta đã gọi trước đó đã dùng các hàm này. Tùy thuộc vào tác vụ được thực hiện, có thể cần gọi `setutxent()` một lần nữa tại các điểm thích hợp sau trong chương trình.

Hàm `setutxent()` và các hàm `getutx*()` mở file utmp nếu chưa mở. Khi ta đã dùng xong file, ta có thể đóng nó bằng hàm `endutxent()`.

```c
#include <utmpx.h>
void endutxent(void);
```

Các hàm `getutxent()`, `getutxid()`, và `getutxline()` đọc một bản ghi từ file utmp và trả về pointer đến structure `utmpx` (được cấp phát tĩnh).

```c
#include <utmpx.h>
struct utmpx *getutxent(void);
struct utmpx *getutxid(const struct utmpx *ut);
struct utmpx *getutxline(const struct utmpx *ut);
                     All return a pointer to a statically allocated utmpx structure,
                          or NULL if no matching record or EOF was encountered
```

Hàm `getutxent()` lấy bản ghi tiếp theo theo tuần tự từ file utmp. Các hàm `getutxid()` và `getutxline()` thực hiện tìm kiếm, bắt đầu từ vị trí file hiện tại, cho bản ghi khớp với tiêu chí được chỉ định trong structure `utmpx` được trỏ bởi đối số `ut`.

Hàm `getutxid()` tìm kiếm file utmp cho bản ghi dựa trên các giá trị được chỉ định trong các trường `ut_type` và `ut_id` của đối số `ut`:

- Nếu trường `ut_type` là `RUN_LVL`, `BOOT_TIME`, `NEW_TIME`, hoặc `OLD_TIME`, thì `getutxid()` tìm bản ghi tiếp theo có trường `ut_type` khớp với giá trị được chỉ định. (Các bản ghi kiểu này không tương ứng với lần đăng nhập của người dùng.) Điều này cho phép tìm kiếm bản ghi về thay đổi system time và run-level.
- Nếu trường `ut_type` là một trong các giá trị hợp lệ còn lại (`INIT_PROCESS`, `LOGIN_PROCESS`, `USER_PROCESS`, hoặc `DEAD_PROCESS`), thì `getutxent()` tìm bản ghi tiếp theo có trường `ut_type` khớp với bất kỳ giá trị nào trong số này và có trường `ut_id` khớp với giá trị chỉ định trong đối số `ut`. Điều này cho phép quét file cho các bản ghi tương ứng với một terminal cụ thể.

Hàm `getutxline()` tìm kiếm về phía trước bản ghi có trường `ut_type` là `LOGIN_PROCESS` hoặc `USER_PROCESS`, và có trường `ut_line` khớp với giá trị chỉ định trong đối số `ut`. Điều này hữu ích để tìm bản ghi tương ứng với lần đăng nhập người dùng.

Cả `getutxid()` lẫn `getutxline()` trả về `NULL` nếu tìm kiếm thất bại (tức là gặp end-of-file mà không tìm thấy bản ghi khớp).

Trên một số triển khai UNIX, `getutxline()` và `getutxid()` xử lý vùng tĩnh dùng để trả về structure `utmpx` như một loại cache. Nếu chúng xác định rằng bản ghi được đặt trong cache này bởi lời gọi `getutx*()` trước đó khớp với tiêu chí được chỉ định trong `ut`, thì không thực hiện đọc file; lời gọi chỉ đơn giản trả về cùng bản ghi đó một lần nữa (SUSv3 cho phép hành vi này). Do đó, để ngăn chặn việc cùng bản ghi được trả về nhiều lần khi gọi `getutxline()` và `getutxid()` trong vòng lặp, ta phải xóa cấu trúc dữ liệu tĩnh này bằng code như sau:

```c
struct utmpx *res = NULL;
/* Other code omitted */
if (res != NULL) /* If 'res' was set via a previous call */
    memset(res, 0, sizeof(struct utmpx));
res = getutxline(&ut);
```

Triển khai glibc không thực hiện loại caching này, nhưng ta vẫn nên dùng kỹ thuật này để đảm bảo portability.

> Vì các hàm `getutx*()` trả về pointer đến structure được cấp phát tĩnh, chúng không phải là reentrant. GNU C library cung cấp các phiên bản reentrant của các hàm utmp truyền thống (`getutent_r()`, `getutid_r()`, và `getutline_r()`), nhưng không cung cấp phiên bản reentrant của các hàm utmpx tương ứng. (SUSv3 không quy định các phiên bản reentrant.)

Theo mặc định, tất cả các hàm `getutx*()` hoạt động trên file utmp chuẩn. Nếu ta muốn dùng file khác, như file wtmp, ta phải gọi `utmpxname()` trước, chỉ định pathname mong muốn.

```c
#define _GNU_SOURCE
#include <utmpx.h>
int utmpxname(const char *file);
                                              Returns 0 on success, or –1 on error
```

Hàm `utmpxname()` chỉ đơn thuần ghi lại bản sao pathname được cung cấp cho nó. Nó không mở file, nhưng đóng mọi file đã được mở trước đó bởi một trong các lời gọi khác. Điều này có nghĩa là `utmpxname()` không trả về lỗi nếu pathname không hợp lệ được chỉ định. Thay vào đó, khi một trong các hàm `getutx*()` được gọi sau đó, nó sẽ trả về lỗi (tức là `NULL`, với `errno` được đặt thành `ENOENT`) khi không mở được file.

> Mặc dù không được quy định trong SUSv3, hầu hết các triển khai UNIX cung cấp `utmpxname()` hoặc hàm `utmpname()` tương đương.

### Chương trình ví dụ

Chương trình trong Listing 40-2 dùng một số hàm mô tả trong mục này để dump nội dung file định dạng utmpx. Session log shell sau minh họa kết quả khi dùng chương trình này để dump nội dung của `/var/run/utmp`:

```
$ ./dump_utmpx
user     type      PID  line   id  host  date/time
LOGIN    LOGIN_PR  1761 tty1   1         Sat Oct 23 09:29:37 2010
LOGIN    LOGIN_PR  1762 tty2   2         Sat Oct 23 09:29:37 2010
lynley   USER_PR  10482 tty3   3         Sat Oct 23 10:19:43 2010
david    USER_PR   9664 tty4   4         Sat Oct 23 10:07:50 2010
liz      USER_PR   1985 tty5   5         Sat Oct 23 10:50:12 2010
mtk      USER_PR  10111 pts/0 /0         Sat Oct 23 09:30:57 2010
```

Để ngắn gọn, chúng ta đã bỏ bớt nhiều kết quả được tạo ra bởi chương trình. Các dòng khớp với `tty1` đến `tty5` là cho các lần đăng nhập trên virtual console (`/dev/tty[1-6]`). Dòng cuối là cho một phiên xterm trên pseudoterminal.

Kết quả sau được tạo ra khi dump `/var/log/wtmp` cho thấy khi người dùng đăng nhập và đăng xuất, hai bản ghi được ghi vào file wtmp. Bằng cách tìm kiếm tuần tự qua file wtmp (dùng `getutxline()`), các bản ghi này có thể được khớp qua trường `ut_line`.

```
$ ./dump_utmpx /var/log/wtmp
user     type      PID  line  id  host        date/time
lynley   USER_PR  10482 tty3  3               Sat Oct 23 10:19:43 2010
         DEAD_PR  10482 tty3  3   2.4.20-4G   Sat Oct 23 10:32:54 2010
```

**Listing 40-2:** Hiển thị nội dung của file định dạng utmpx

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––– loginacct/dump_utmpx.c
#define _GNU_SOURCE
#include <time.h>
#include <utmpx.h>
#include <paths.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    struct utmpx *ut;
    if (argc > 1 && strcmp(argv[1], "--help") == 0)
        usageErr("%s [utmp-pathname]\n", argv[0]);
    if (argc > 1)               /* Use alternate file if supplied */
        if (utmpxname(argv[1]) == -1)
            errExit("utmpxname");
    setutxent();
    printf("user     type      PID  line   id  host  date/time\n");
    while ((ut = getutxent()) != NULL) {    /* Sequential scan to EOF */
        printf("%-8s ", ut->ut_user);
        printf("%-9.9s ",
            (ut->ut_type == EMPTY)         ? "EMPTY" :
            (ut->ut_type == RUN_LVL)       ? "RUN_LVL" :
            (ut->ut_type == BOOT_TIME)     ? "BOOT_TIME" :
            (ut->ut_type == NEW_TIME)      ? "NEW_TIME" :
            (ut->ut_type == OLD_TIME)      ? "OLD_TIME" :
            (ut->ut_type == INIT_PROCESS)  ? "INIT_PR" :
            (ut->ut_type == LOGIN_PROCESS) ? "LOGIN_PR" :
            (ut->ut_type == USER_PROCESS)  ? "USER_PR" :
            (ut->ut_type == DEAD_PROCESS)  ? "DEAD_PR" : "???");
        printf("%5ld %-6.6s %-3.5s %-9.9s ", (long) ut->ut_pid,
               ut->ut_line, ut->ut_id, ut->ut_host);
        printf("%s", ctime((time_t *) &(ut->ut_tv.tv_sec)));
    }
    endutxent();
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––– loginacct/dump_utmpx.c
```

---

## 40.5 Lấy Tên Đăng Nhập: getlogin()

Hàm `getlogin()` trả về tên người dùng đã đăng nhập trên controlling terminal của process gọi. Hàm này dùng thông tin được lưu trong file utmp.

```c
#include <unistd.h>
char *getlogin(void);
                            Returns pointer to username string, or NULL on error
```

Hàm `getlogin()` gọi `ttyname()` (Mục 62.10) để tìm tên terminal liên kết với standard input của process gọi. Sau đó nó tìm kiếm file utmp cho bản ghi có giá trị `ut_line` khớp với tên terminal này. Nếu tìm thấy bản ghi khớp, `getlogin()` trả về chuỗi `ut_user` từ bản ghi đó.

Nếu không tìm thấy khớp hoặc xảy ra lỗi, `getlogin()` trả về `NULL` và đặt `errno` để chỉ ra lỗi. Một lý do `getlogin()` có thể thất bại là process không có terminal liên kết với standard input (`ENOTTY`), có thể vì nó là daemon. Một khả năng khác là phiên terminal này không được ghi trong utmp; ví dụ, một số terminal emulator phần mềm không tạo mục trong file utmp.

Ngay cả trong trường hợp (bất thường) khi một user ID có nhiều tên đăng nhập trong `/etc/passwd`, `getlogin()` vẫn có thể trả về username thực sự đã được dùng để đăng nhập trên terminal này vì nó dựa vào file utmp. Ngược lại, dùng `getpwuid(getuid())` luôn lấy bản ghi khớp đầu tiên từ `/etc/passwd`, bất kể tên nào đã được dùng để đăng nhập.

> Phiên bản reentrant của `getlogin()` được quy định bởi SUSv3, dưới dạng `getlogin_r()`, và hàm này được cung cấp bởi glibc.

> Biến môi trường `LOGNAME` cũng có thể được dùng để tìm tên đăng nhập của người dùng. Tuy nhiên, giá trị của biến này có thể bị người dùng thay đổi, nghĩa là nó không thể được dùng để xác định an toàn danh tính người dùng.

---

## 40.6 Cập Nhật Các File utmp và wtmp cho Login Session

Khi viết ứng dụng tạo login session (theo cách của `login` hay `sshd`), ta nên cập nhật các file utmp và wtmp như sau:

- **Khi đăng nhập**, một bản ghi nên được ghi vào file utmp để chỉ ra rằng người dùng này đã đăng nhập. Ứng dụng phải kiểm tra xem bản ghi cho terminal này đã tồn tại trong file utmp chưa. Nếu có bản ghi trước đó, nó sẽ bị ghi đè; nếu không, một bản ghi mới được nối vào file. Thường thì gọi `pututxline()` (mô tả ngay sau) là đủ để đảm bảo các bước này được thực hiện đúng (xem Listing 40-3 để có ví dụ). Bản ghi `utmpx` đầu ra phải có ít nhất các trường `ut_type`, `ut_user`, `ut_tv`, `ut_pid`, `ut_id`, và `ut_line` được điền. Trường `ut_type` phải được đặt thành `USER_PROCESS`. Trường `ut_id` phải chứa hậu tố của tên thiết bị (tức là terminal hoặc pseudoterminal) mà người dùng đang đăng nhập vào, và trường `ut_line` phải chứa tên thiết bị đăng nhập với phần đầu `/dev/` bị xóa. Một bản ghi chứa đúng cùng thông tin được nối vào file wtmp.

  > Tên terminal đóng vai trò (qua các trường `ut_line` và `ut_id`) như khóa duy nhất cho các bản ghi trong file utmp.

- **Khi đăng xuất**, bản ghi đã ghi vào file utmp trước đó nên được xóa. Điều này được thực hiện bằng cách tạo một bản ghi với `ut_type` được đặt thành `DEAD_PROCESS`, và có cùng giá trị `ut_id` và `ut_line` như bản ghi được ghi trong quá trình đăng nhập, nhưng với trường `ut_user` được đặt về không. Bản ghi này được ghi đè lên bản ghi trước đó. Một bản sao của cùng bản ghi được nối vào file wtmp.

  > Nếu ta không dọn sạch bản ghi utmp khi đăng xuất, có thể vì chương trình bị crash, thì khi khởi động lại, `init` tự động dọn sạch bản ghi, đặt `ut_type` thành `DEAD_PROCESS` và xóa các trường khác của bản ghi.

Các file utmp và wtmp thường được bảo vệ để chỉ những người dùng có đặc quyền mới có thể thực hiện cập nhật trên các file này. Độ chính xác của `getlogin()` phụ thuộc vào tính toàn vẹn của file utmp. Vì lý do này và các lý do khác, các quyền trên các file utmp và wtmp không bao giờ nên được đặt để cho phép người dùng không có đặc quyền ghi.

Điều gì cấu thành một login session? Như ta có thể mong đợi, các lần đăng nhập qua `login`, `telnet`, và `ssh` được ghi trong các file login accounting. Hầu hết các triển khai `ftp` cũng tạo bản ghi login accounting. Tuy nhiên, liệu bản ghi login accounting có được tạo cho mỗi terminal window được khởi động trên hệ thống hay cho các lời gọi `su` không? Câu trả lời cho câu hỏi đó thay đổi giữa các triển khai UNIX.

> Trong một số chương trình terminal emulator (ví dụ: xterm), các tùy chọn command-line hoặc các cơ chế khác có thể được dùng để xác định liệu chương trình có cập nhật các file login accounting hay không.

Hàm `pututxline()` ghi structure `utmpx` được trỏ bởi `ut` vào file `/var/run/utmp` (hoặc file thay thế nếu `utmpxname()` đã được gọi trước đó).

```c
#include <utmpx.h>
struct utmpx *pututxline(const struct utmpx *ut);
             Returns pointer to copy of successfully updated record on success,
                                                                  or NULL on error
```

Trước khi ghi bản ghi, `pututxline()` đầu tiên dùng `getutxid()` để tìm kiếm về phía trước bản ghi có thể bị ghi đè. Nếu tìm thấy bản ghi như vậy, nó sẽ bị ghi đè; nếu không, một bản ghi mới được nối vào cuối file. Trong nhiều trường hợp, ứng dụng gọi `pututxline()` trước bằng một lời gọi đến một trong các hàm `getutx*()`, đặt vị trí file hiện tại về bản ghi đúng — tức là bản ghi khớp với tiêu chí kiểu `getutxid` trong structure `utmpx` được trỏ bởi `ut`. Nếu `pututxline()` xác định điều này đã xảy ra, nó không gọi `getutxid()`.

> Nếu `pututxline()` thực hiện lời gọi nội bộ đến `getutxid()`, lời gọi này không thay đổi vùng tĩnh được các hàm `getutx*()` dùng để trả về structure `utmpx`. SUSv3 yêu cầu hành vi này từ một triển khai.

Khi cập nhật file wtmp, ta đơn giản mở file và nối một bản ghi vào nó. Vì đây là thao tác chuẩn, glibc đóng gói nó trong hàm `updwtmpx()`.

```c
#define _GNU_SOURCE
#include <utmpx.h>
void updwtmpx(char *wtmpx_file, struct utmpx *ut);
```

Hàm `updwtmpx()` nối bản ghi `utmpx` được trỏ bởi `ut` vào file được chỉ định trong `wtmpx_file`.

SUSv3 không quy định `updwtmpx()`, và nó chỉ xuất hiện trên một số triển khai UNIX khác. Các triển khai khác cung cấp các hàm liên quan — `login(3)`, `logout(3)`, và `logwtmp(3)` — cũng có trong glibc và được mô tả trong manual page. Nếu các hàm như vậy không có, ta cần viết các hàm tương đương của mình. (Việc triển khai các hàm này không phức tạp.)

### Chương trình ví dụ

Listing 40-3 dùng các hàm mô tả trong mục này để cập nhật các file utmp và wtmp. Chương trình này thực hiện các cập nhật cần thiết cho utmp và wtmp để đăng nhập người dùng có tên trên command line, và sau đó, sau khi sleep vài giây, đăng xuất họ lại. Thông thường, các hành động như vậy sẽ liên quan đến việc tạo và kết thúc login session cho người dùng. Chương trình này dùng `ttyname()` để lấy tên thiết bị terminal liên kết với file descriptor. Chúng ta mô tả `ttyname()` trong Mục 62.10.

Session log shell sau minh họa hoạt động của chương trình trong Listing 40-3. Chúng ta giả định đặc quyền để có thể cập nhật các file login accounting, rồi dùng chương trình để tạo bản ghi cho người dùng `mtk`:

```
$ su
Password:
# ./utmpx_login mtk
Creating login entries in utmp and wtmp
  using pid 1471, line pts/7, id /7
Type Control-Z to suspend program
[1]+ Stopped    ./utmpx_login mtk
```

Trong khi chương trình `utmpx_login` đang sleep, ta gõ Control-Z để tạm dừng chương trình và đẩy nó vào background. Tiếp theo, ta dùng chương trình trong Listing 40-2 để kiểm tra nội dung file utmp:

```
# ./dump_utmpx /var/run/utmp
user     type      PID  line   id  host  date/time
cecilia  USER_PR   249  tty1   1         Fri Feb  1 21:39:07 2008
mtk      USER_PR  1471  pts/7 /7         Fri Feb  1 22:08:06 2008
# who
cecilia  tty1    Feb  1 21:39
mtk      pts/7   Feb  1 22:08
```

Kết quả `who(1)` cho thấy kết quả của `who` xuất phát từ utmp.

Tiếp theo ta dùng chương trình để kiểm tra nội dung file wtmp:

```
# ./dump_utmpx /var/log/wtmp
user     type      PID  line   id  host  date/time
cecilia  USER_PR   249  tty1   1         Fri Feb  1 21:39:07 2008
mtk      USER_PR  1471  pts/7 /7         Fri Feb  1 22:08:06 2008
# last mtk
mtk  pts/7    Fri Feb  1 22:08   still logged in
```

Lệnh `last(1)` cho thấy kết quả của `last` xuất phát từ wtmp.

Tiếp theo, ta dùng lệnh `fg` để tiếp tục chương trình `utmpx_login` ở foreground. Sau đó nó ghi các bản ghi đăng xuất vào file utmp và wtmp.

```
# fg
./utmpx_login mtk
Creating logout entries in utmp and wtmp
```

Ta sau đó kiểm tra lại nội dung file utmp. Ta thấy bản ghi utmp đã bị ghi đè:

```
# ./dump_utmpx /var/run/utmp
user     type      PID  line   id  host  date/time
cecilia  USER_PR   249  tty1   1         Fri Feb  1 21:39:07 2008
         DEAD_PR  1471  pts/7 /7         Fri Feb  1 22:09:09 2008
# who
cecilia  tty1    Feb  1 21:39
```

Dòng cuối kết quả cho thấy `who` đã bỏ qua bản ghi `DEAD_PROCESS`.

Khi ta kiểm tra file wtmp, ta thấy bản ghi wtmp đã được bổ sung:

```
# ./dump_utmpx /var/log/wtmp
user     type      PID  line   id  host  date/time
cecilia  USER_PR   249  tty1   1         Fri Feb  1 21:39:07 2008
mtk      USER_PR  1471  pts/7 /7         Fri Feb  1 22:08:06 2008
         DEAD_PR  1471  pts/7 /7         Fri Feb  1 22:09:09 2008
# last mtk
mtk  pts/7    Fri Feb  1 22:08 - 22:09  (00:01)
```

Dòng cuối kết quả trên cho thấy `last` khớp các bản ghi đăng nhập và đăng xuất trong wtmp để hiển thị thời gian bắt đầu và kết thúc của login session đã hoàn thành.

**Listing 40-3:** Cập nhật các file utmp và wtmp

```c
––––––––––––––––––––––––––––––––––––––––––––––––––– loginacct/utmpx_login.c
#define _GNU_SOURCE
#include <time.h>
#include <utmpx.h>
#include <paths.h>  /* Definitions of _PATH_UTMP and _PATH_WTMP */
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    struct utmpx ut;
    char *devName;
    if (argc < 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s username [sleep-time]\n", argv[0]);
    /* Initialize login record for utmp and wtmp files */
    memset(&ut, 0, sizeof(struct utmpx));
    ut.ut_type = USER_PROCESS;              /* This is a user login */
    strncpy(ut.ut_user, argv[1], sizeof(ut.ut_user));
    if (time((time_t *) &ut.ut_tv.tv_sec) == -1)
        errExit("time");                    /* Stamp with current time */
    ut.ut_pid = getpid();
    /* Set ut_line and ut_id based on the terminal associated with
       'stdin'. This code assumes terminals named "/dev/[pt]t[sy]*".
       The "/dev/" dirname is 5 characters; the "[pt]t[sy]" filename
       prefix is 3 characters (making 8 characters in all). */
    devName = ttyname(STDIN_FILENO);
    if (devName == NULL)
        errExit("ttyname");
    if (strlen(devName) <= 8)               /* Should never happen */
        fatal("Terminal name is too short: %s", devName);
    strncpy(ut.ut_line, devName + 5, sizeof(ut.ut_line));
    strncpy(ut.ut_id,   devName + 8, sizeof(ut.ut_id));
    printf("Creating login entries in utmp and wtmp\n");
    printf("  using pid %ld, line %.*s, id %.*s\n",
           (long) ut.ut_pid, (int) sizeof(ut.ut_line), ut.ut_line,
           (int) sizeof(ut.ut_id), ut.ut_id);
    setutxent();                            /* Rewind to start of utmp file */
    if (pututxline(&ut) == NULL)            /* Write login record to utmp */
        errExit("pututxline");
    updwtmpx(_PATH_WTMP, &ut);              /* Append login record to wtmp */
    /* Sleep a while, so we can examine utmp and wtmp files */
    sleep((argc > 2) ? getInt(argv[2], GN_NONNEG, "sleep-time") : 15);
    /* Now do a "logout"; use values from previously initialized 'ut',
       except for changes below */
    ut.ut_type = DEAD_PROCESS;              /* Required for logout record */
    time((time_t *) &ut.ut_tv.tv_sec);      /* Stamp with logout time */
    memset(&ut.ut_user, 0, sizeof(ut.ut_user));
                                            /* Logout record has null username */
    printf("Creating logout entries in utmp and wtmp\n");
    setutxent();                            /* Rewind to start of utmp file */
    if (pututxline(&ut) == NULL)            /* Overwrite previous utmp record */
        errExit("pututxline");
    updwtmpx(_PATH_WTMP, &ut);              /* Append logout record to wtmp */
    endutxent();
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––– loginacct/utmpx_login.c
```

---

## 40.7 File lastlog

File lastlog ghi lại thời gian mỗi người dùng đăng nhập lần cuối vào hệ thống. (Điều này khác với file wtmp, ghi lại tất cả các lần đăng nhập và đăng xuất của tất cả người dùng.) Trong số những thứ khác, file lastlog cho phép chương trình `login` thông báo cho người dùng (khi bắt đầu login session mới) khi họ đăng nhập lần cuối. Ngoài việc cập nhật utmp và wtmp, các ứng dụng cung cấp dịch vụ đăng nhập cũng nên cập nhật lastlog.

Như với các file utmp và wtmp, có sự biến thể về vị trí và định dạng của file lastlog. (Một số triển khai UNIX không cung cấp file này.) Trên Linux, file này nằm tại `/var/log/lastlog`, và hằng số `_PATH_LASTLOG` được định nghĩa trong `<paths.h>` để trỏ đến vị trí này. Như các file utmp và wtmp, file lastlog thường được bảo vệ để tất cả người dùng có thể đọc nhưng chỉ các process có đặc quyền mới có thể cập nhật.

Các bản ghi trong file lastlog có định dạng sau (được định nghĩa trong `<lastlog.h>`):

```c
#define UT_NAMESIZE 32
#define UT_HOSTSIZE 256
struct lastlog {
    time_t ll_time;               /* Time of last login */
    char   ll_line[UT_NAMESIZE];  /* Terminal for remote login */
    char   ll_host[UT_HOSTSIZE];  /* Hostname for remote login */
};
```

Lưu ý rằng các bản ghi này không bao gồm username hay user ID. Thay vào đó, file lastlog bao gồm một loạt các bản ghi được đánh chỉ mục bởi user ID. Do đó, để tìm bản ghi lastlog cho user ID 1000, ta sẽ seek đến byte `(1000 * sizeof(struct lastlog))` của file. Điều này được minh họa trong Listing 40-4, một chương trình cho phép ta xem các bản ghi lastlog cho người dùng được liệt kê trên command line. Ví dụ kết quả:

```
$ ./view_lastlog annie paulh
annie    tty2         Mon Jan 17 11:00:12 2011
paulh    pts/11       Sat Aug 14 09:22:14 2010
```

Thực hiện cập nhật lastlog cũng tương tự là việc mở file, seek đến vị trí đúng, và thực hiện ghi.

Vì file lastlog được đánh chỉ mục bởi user ID, không thể phân biệt các lần đăng nhập dưới các username khác nhau có cùng user ID. (Trong Mục 8.1, chúng ta đã nói rằng có thể, mặc dù không phổ biến, có nhiều tên đăng nhập với cùng user ID.)

**Listing 40-4:** Hiển thị thông tin từ file lastlog

```c
–––––––––––––––––––––––––––––––––––––––––––––––––– loginacct/view_lastlog.c
#include <time.h>
#include <lastlog.h>
#include <paths.h>          /* Definition of _PATH_LASTLOG */
#include <fcntl.h>
#include "ugid_functions.h" /* Declaration of userIdFromName() */
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    struct lastlog llog;
    int fd, j;
    uid_t uid;
    if (argc > 1 && strcmp(argv[1], "--help") == 0)
        usageErr("%s [username...]\n", argv[0]);
    fd = open(_PATH_LASTLOG, O_RDONLY);
    if (fd == -1)
        errExit("open");
    for (j = 1; j < argc; j++) {
        uid = userIdFromName(argv[j]);
        if (uid == -1) {
            printf("No such user: %s\n", argv[j]);
            continue;
        }
        if (lseek(fd, uid * sizeof(struct lastlog), SEEK_SET) == -1)
            errExit("lseek");
        if (read(fd, &llog, sizeof(struct lastlog)) <= 0) {
            printf("read failed for %s\n", argv[j]); /* EOF or error */
            continue;
        }
        printf("%-8.8s %-6.6s %-20.20s %s", argv[j], llog.ll_line,
               llog.ll_host, ctime((time_t *) &llog.ll_time));
    }
    close(fd);
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––– loginacct/view_lastlog.c
```

---

## 40.8 Tóm Tắt

Login accounting ghi lại các người dùng hiện đang đăng nhập, cũng như tất cả các lần đăng nhập trong quá khứ. Thông tin này được duy trì trong ba file: file utmp, lưu bản ghi về tất cả người dùng hiện đang đăng nhập; file wtmp, là audit trail của tất cả các lần đăng nhập và đăng xuất; và file lastlog, ghi lại thời gian đăng nhập lần cuối cho mỗi người dùng. Nhiều lệnh, như `who` và `last`, dùng thông tin trong các file này.

Thư viện C cung cấp các hàm để lấy và cập nhật thông tin trong các file login accounting. Các ứng dụng cung cấp dịch vụ đăng nhập nên dùng các hàm này để cập nhật các file login accounting, để các lệnh phụ thuộc vào thông tin này hoạt động chính xác.

---

## 40.9 Bài Tập

- **40-1.** Triển khai `getlogin()`. Như đã nói trong Mục 40.5, `getlogin()` có thể không hoạt động đúng với các process chạy dưới một số terminal emulator phần mềm; trong trường hợp đó, hãy kiểm tra từ virtual console.
- **40-2.** Sửa đổi chương trình trong Listing 40-3 (`utmpx_login.c`) để nó cũng cập nhật file lastlog ngoài các file utmp và wtmp.
- **40-3.** Đọc manual page cho `login(3)`, `logout(3)`, và `logwtmp(3)`. Triển khai các hàm này.
- **40-4.** Triển khai phiên bản đơn giản của `who(1)`.
