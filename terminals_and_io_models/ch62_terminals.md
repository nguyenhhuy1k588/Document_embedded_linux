## Chương 62
# Terminal

Ngày xưa, người dùng truy cập hệ thống UNIX qua terminal kết nối bằng đường serial. Ngày nay, terminal thực sự đã được thay thế bởi terminal emulator và virtual console. Tuy nhiên, việc lập trình terminal vẫn cần thiết khi làm việc với terminal emulator (dùng pseudoterminal), virtual console, và thiết bị kết nối qua serial line.

---

## 62.1 Tổng Quan

Cả terminal thực và terminal emulator đều có terminal driver liên quan xử lý input và output. Terminal driver có thể hoạt động ở hai mode input:

- **Canonical mode**: Input được xử lý theo dòng, line editing được bật. Đây là mode mặc định.
- **Noncanonical mode**: Input không được tập hợp thành dòng; có sẵn ngay lập tức. Dùng trong các chương trình như `vi`, `more`.

Terminal driver duy trì hai queue: input queue (ký tự từ terminal đến process) và output queue (ký tự từ process đến terminal). Khi terminal echoing được bật, terminal driver tự động thêm bản sao ký tự input vào cuối output queue.

---

## 62.2 Lấy và Thay Đổi Terminal Attribute

Hai hàm `tcgetattr()` và `tcsetattr()` lấy và thay đổi attribute của terminal.

```c
#include <termios.h>
int tcgetattr(int fd, struct termios *termios_p);
int tcsetattr(int fd, int optional_actions, const struct termios *termios_p);
                                         Both return 0 on success, or –1 on error
```

Structure `termios`:

```c
struct termios {
    tcflag_t c_iflag;   /* Input flags */
    tcflag_t c_oflag;   /* Output flags */
    tcflag_t c_cflag;   /* Control flags */
    tcflag_t c_lflag;   /* Local modes */
    cc_t     c_line;    /* Line discipline */
    cc_t     c_cc[NCCS]; /* Terminal special characters */
    speed_t  c_ispeed;  /* Input speed (unused) */
    speed_t  c_ospeed;  /* Output speed (unused) */
};
```

Đối số `optional_actions` của `tcsetattr()`:
- **`TCSANOW`**: Thay đổi ngay lập tức.
- **`TCSADRAIN`**: Thay đổi sau khi output đã được truyền.
- **`TCSAFLUSH`**: Như `TCSADRAIN`, nhưng còn loại bỏ pending input.

Cách điển hình để thay đổi attribute: lấy settings hiện tại, thay đổi, đẩy lại.

Ví dụ — tắt terminal echoing:
```c
struct termios tp;
if (tcgetattr(STDIN_FILENO, &tp) == -1)  errExit("tcgetattr");
tp.c_lflag &= ~ECHO;
if (tcsetattr(STDIN_FILENO, TCSAFLUSH, &tp) == -1)  errExit("tcsetattr");
```

---

## 62.3 Lệnh stty

Lệnh `stty` là tương tự command-line của `tcgetattr()` và `tcsetattr()`, cho phép xem và thay đổi terminal attribute từ shell.

```bash
$ stty -a                  # Hiển thị tất cả settings
$ stty intr ^L             # Đặt interrupt character thành Control-L
$ stty tostop              # Bật TOSTOP flag
$ stty sane                # Khôi phục terminal về trạng thái "sane"
```

Nếu terminal bị để ở trạng thái không dùng được, gõ: `Control-J stty sane Control-J`

---

## 62.4 Terminal Special Character

Các ký tự đặc biệt của terminal được lưu trong mảng `c_cc` của structure `termios`. Một số ký tự quan trọng:

**Bảng 62-1:** Terminal special character

| Ký tự | Subscript c_cc | Mô tả | Mặc định |
|-------|---------------|-------|---------|
| CR | (none) | Carriage return | ^M |
| EOF | VEOF | End-of-file | ^D |
| ERASE | VERASE | Xóa ký tự trước | ^? |
| INTR | VINTR | Interrupt (SIGINT) | ^C |
| KILL | VKILL | Xóa dòng | ^U |
| LNEXT | VLNEXT | Ký tự tiếp theo theo nghĩa đen | ^V |
| NL | (none) | Newline | ^J |
| QUIT | VQUIT | Quit (SIGQUIT) | ^\\ |
| START | VSTART | Bắt đầu output | ^Q |
| STOP | VSTOP | Dừng output | ^S |
| SUSP | VSUSP | Suspend (SIGTSTP) | ^Z |

Ký tự đặc biệt có thể bị vô hiệu hóa bằng cách đặt giá trị `fpathconf(fd, _PC_VDISABLE)`.

---

## 62.5 Terminal Flag

Bốn trường flag trong `termios`:

**c_iflag** (input): `BRKINT`, `ICRNL` (map CR→NL, mặc định on), `IGNBRK`, `INLCR`, `IXON` (flow control, mặc định on), `ISTRIP`, v.v.

**c_oflag** (output): `OPOST` (output postprocessing, mặc định on), `ONLCR` (map NL→CR-NL, mặc định on), `OCRNL`, v.v.

**c_cflag** (hardware): `CREAD` (allow input), `CSIZE` (character size), `PARENB`, `HUPCL`, v.v.

**c_lflag** (local): `ECHO` (on), `ECHOE` (visual ERASE, on), `ICANON` (canonical mode, on), `ISIG` (signal characters, on), `IEXTEN` (extended processing, on), `TOSTOP`, v.v.

Một số flag quan trọng:
- **`ECHO`**: Bật echoing input. Tắt khi đọc mật khẩu.
- **`ICANON`**: Bật canonical mode input.
- **`ISIG`**: Bật signal-generating character (INTR, QUIT, SUSP).
- **`TOSTOP`**: Tạo `SIGTTOU` cho background output.
- **`OPOST`**: Bật output postprocessing.

---

## 62.6 Terminal I/O Mode

### 62.6.1 Canonical Mode

Canonical mode (ICANON được bật): input được tập hợp thành dòng, line editing được bật. `read()` trả về khi có đủ một dòng input.

### 62.6.2 Noncanonical Mode

Noncanonical mode (ICANON tắt): không xử lý đặc biệt input, input sẵn sàng ngay lập tức. Hai yếu tố `MIN` và `TIME` trong `c_cc` kiểm soát khi nào `read()` hoàn thành:

- **MIN=0, TIME=0** (polling read): Trả về ngay — dữ liệu có sẵn hoặc trả về 0.
- **MIN>0, TIME=0** (blocking read): Block cho đến khi có `MIN` byte. Dùng `MIN=1, TIME=0` để đọc từng ký tự.
- **MIN=0, TIME>0** (read with timeout): Trả về khi có ≥1 byte hoặc sau `TIME/10` giây.
- **MIN>0, TIME>0** (read with interbyte timeout): Timer restart sau mỗi byte. Hữu ích để xử lý escape sequence.

> **Lưu ý portable**: `VMIN` và `VEOF` có thể có cùng giá trị trên một số implementation. Khi chuyển sang noncanonical mode, lưu termios settings trước và khôi phục khi trở về canonical mode.

### 62.6.3 Cooked, Cbreak, và Raw Mode

Ba mode lịch sử từ Seventh Edition UNIX:

**Bảng 62-3:** Khác biệt giữa cooked, cbreak, và raw mode

| Tính năng | Cooked | Cbreak | Raw |
|-----------|--------|--------|-----|
| Input | Theo dòng | Từng ký tự | Từng ký tự |
| Line editing | Có | Không | Không |
| Signal character | Có | Có | Không |
| START/STOP | Có | Có | Không |
| Các ký tự đặc biệt khác | Có | Không | Không |
| Output processing | Có | Có | Không |
| Echo | Có | Tùy | Không |

**Listing 62-3:** Chuyển terminal sang cbreak và raw mode

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––tty/tty_functions.c
#include <termios.h>
#include <unistd.h>
#include "tty_functions.h"
int
ttySetCbreak(int fd, struct termios *prevTermios)
{
    struct termios t;
    if (tcgetattr(fd, &t) == -1)  return -1;
    if (prevTermios != NULL)  *prevTermios = t;
    t.c_lflag &= ~(ICANON | ECHO);
    t.c_lflag |= ISIG;
    t.c_iflag &= ~ICRNL;
    t.c_cc[VMIN] = 1;
    t.c_cc[VTIME] = 0;
    if (tcsetattr(fd, TCSAFLUSH, &t) == -1)  return -1;
    return 0;
}
int
ttySetRaw(int fd, struct termios *prevTermios)
{
    struct termios t;
    if (tcgetattr(fd, &t) == -1)  return -1;
    if (prevTermios != NULL)  *prevTermios = t;
    t.c_lflag &= ~(ICANON | ISIG | IEXTEN | ECHO);
    t.c_iflag &= ~(BRKINT | ICRNL | IGNBRK | IGNCR | INLCR |
                   INPCK | ISTRIP | IXON | PARMRK);
    t.c_oflag &= ~OPOST;
    t.c_cc[VMIN] = 1;
    t.c_cc[VTIME] = 0;
    if (tcsetattr(fd, TCSAFLUSH, &t) == -1)  return -1;
    return 0;
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––tty/tty_functions.c
```

Chương trình sử dụng raw/cbreak mode phải khôi phục terminal về mode bình thường khi kết thúc, kể cả khi nhận signal. Xem Listing 62-4 trong source code.

---

## 62.7 Terminal Line Speed (Bit Rate)

Các hàm lấy/đặt line speed:

```c
#include <termios.h>
speed_t cfgetispeed(const struct termios *termios_p);
speed_t cfgetospeed(const struct termios *termios_p);
int     cfsetospeed(struct termios *termios_p, speed_t speed);
int     cfsetispeed(struct termios *termios_p, speed_t speed);
```

Các hằng số speed: `B300`, `B2400`, `B9600`, `B38400`, v.v.

Ví dụ đặt output speed thành 38400 bps:
```c
struct termios tp;
tcgetattr(fd, &tp);
cfsetospeed(&tp, B38400);
tcsetattr(fd, TCSAFLUSH, &tp);
```

---

## 62.8 Terminal Line Control

Các hàm thực hiện line control:

```c
#include <termios.h>
int tcsendbreak(int fd, int duration); /* Tạo BREAK condition */
int tcdrain(int fd);                   /* Block cho đến khi output xong */
int tcflush(int fd, int queue_selector); /* Flush input/output queue */
int tcflow(int fd, int action);         /* Kiểm soát flow */
```

**Giá trị `queue_selector` của `tcflush()`**:
| Giá trị | Mô tả |
|---------|-------|
| `TCIFLUSH` | Flush input queue |
| `TCOFLUSH` | Flush output queue |
| `TCIOFLUSH` | Flush cả hai |

**Giá trị `action` của `tcflow()`**:
| Giá trị | Mô tả |
|---------|-------|
| `TCOOFF` | Tạm dừng output đến terminal |
| `TCOON` | Resume output đến terminal |
| `TCIOFF` | Truyền ký tự STOP đến terminal |
| `TCION` | Truyền ký tự START đến terminal |

---

## 62.9 Terminal Window Size

Kernel hỗ trợ giám sát kích thước terminal window:
- Signal `SIGWINCH` gửi đến foreground process group khi window thay đổi.
- `ioctl(fd, TIOCGWINSZ, &ws)` lấy kích thước hiện tại.

```c
struct winsize {
    unsigned short ws_row;    /* Số hàng (ký tự) */
    unsigned short ws_col;    /* Số cột (ký tự) */
    unsigned short ws_xpixel; /* Kích thước ngang (pixel) */
    unsigned short ws_ypixel; /* Kích thước dọc (pixel) */
};
```

Đặt kích thước window: `ioctl(fd, TIOCSWINSZ, &ws)` — nếu khác kích thước hiện tại, `SIGWINCH` được gửi đến foreground process group.

---

## 62.10 Nhận Dạng Terminal

```c
#include <unistd.h>
int    isatty(int fd);
                            Returns true (1) if fd is associated with a terminal
char  *ttyname(int fd);
                        Returns pointer to (statically allocated) string containing
                                                  terminal name, or NULL on error
```

`isatty()`: Xác định xem file descriptor có liên kết với terminal không.
`ttyname()`: Trả về tên của thiết bị terminal liên kết. Phiên bản reentrant: `ttyname_r()`.

---

## 62.11 Tóm Tắt

Terminal setting được lưu trong structure `termios` với bốn trường bit-mask và mảng ký tự đặc biệt. `tcgetattr()` và `tcsetattr()` lấy và thay đổi settings.

Terminal driver có thể hoạt động trong canonical mode (input theo dòng, line editing bật) hoặc noncanonical mode (input từng ký tự, kiểm soát bởi MIN và TIME). Cbreak và raw mode là các biến thể lịch sử được mô phỏng bằng cách thay đổi termios.

---

## 62.12 Bài Tập

- **62-1.** Triển khai `isatty()`.
- **62-2.** Triển khai `ttyname()`.
- **62-3.** Triển khai hàm `getpass()` mô tả trong Mục 8.5.
- **62-4.** Viết chương trình hiển thị thông tin liệu terminal stdin đang ở canonical hay noncanonical mode.
