## Chương 64
# Pseudoterminal

Pseudoterminal là thiết bị ảo cung cấp kênh IPC. Một đầu kênh là chương trình kỳ vọng được kết nối với thiết bị terminal. Đầu kia là chương trình điều khiển chương trình terminal-oriented bằng cách dùng kênh để gửi input và đọc output.

---

## 64.1 Tổng Quan

Pseudoterminal giải quyết vấn đề: làm sao cho phép người dùng trên một host vận hành terminal-oriented program (như `vi`) trên host khác qua mạng?

Socket cung cấp một phần cơ chế qua mạng, nhưng ta không thể kết nối stdin/stdout/stderr của terminal-oriented program trực tiếp với socket vì:
1. Terminal-oriented program kỳ vọng được kết nối với terminal để thực hiện các thao tác terminal.
2. Terminal-oriented program kỳ vọng terminal driver xử lý input/output (ví dụ: ký tự Control-D trong canonical mode).
3. Terminal-oriented program phải có controlling terminal.

### Thiết Bị Master và Slave của Pseudoterminal

Pseudoterminal là cặp thiết bị ảo được kết nối: **pseudoterminal master** và **pseudoterminal slave** (pty pair). Chúng cung cấp kênh IPC hai chiều tương tự pipe hai chiều.

Điểm quan trọng: **slave device hoạt động giống terminal chuẩn**. Tất cả thao tác có thể áp dụng cho terminal đều có thể áp dụng cho pseudoterminal slave.

### Cách Programs Dùng Pseudoterminal

Hai program điển hình dùng pseudoterminal:
- stdin/stdout/stderr của terminal-oriented program được kết nối với pseudoterminal slave (cũng là controlling terminal của program).
- Bên kia, **driver program** đóng vai trò proxy cho người dùng, cung cấp input cho terminal-oriented program và đọc output của nó.

**Quy trình thiết lập**:
1. Driver program mở pseudoterminal master device.
2. Driver program gọi `fork()` tạo child process. Child:
   a. Gọi `setsid()` để tạo session mới (mất controlling terminal).
   b. Mở pseudoterminal slave device (trở thành controlling terminal cho child).
   c. Dùng `dup()` để duplicate file descriptor slave thành stdin/stdout/stderr.
   d. Gọi `exec()` để khởi động terminal-oriented program.

**Ví dụ: ssh** — Trên remote host, ssh server là driver program cho pseudoterminal master, và login shell được kết nối với pseudoterminal slave. Server relay ký tự giữa terminal của user trên local host và shell trên remote host.

**Ứng dụng khác**:
- `expect(1)`: Dùng pseudoterminal để điều khiển terminal-oriented program từ script file.
- Terminal emulator như `xterm`.
- `screen(1)`: Multiplexing terminal vật lý giữa nhiều process.
- `script(1)`: Ghi lại input và output trong shell session.

### Pseudoterminal UNIX 98 (System V) và BSD

Linux hỗ trợ cả hai. Chương này tập trung vào UNIX 98 pseudoterminal (chuẩn hóa trong SUSv3).

---

## 64.2 UNIX 98 Pseudoterminal

### 64.2.1 Mở Master: posix_openpt()

```c
#define _XOPEN_SOURCE 600
#include <stdlib.h>
#include <fcntl.h>
int posix_openpt(int flags);
                                Returns file descriptor on success, or –1 on error
```

Tìm và mở pseudoterminal master device chưa dùng. `flags` thường là `O_RDWR` (và có thể `O_NOCTTY`).

Trên Linux, `posix_openpt()` được triển khai như:
```c
int posix_openpt(int flags) { return open("/dev/ptmx", flags); }
```

Gọi `posix_openpt()` cũng tạo slave device file tương ứng trong thư mục `/dev/pts`.

**Giới hạn**: Từ Linux 2.6.4, giới hạn được định nghĩa bởi `/proc/sys/kernel/pty/max` (mặc định 4096).

### 64.2.2 Thay Đổi Ownership và Quyền Slave: grantpt()

```c
#define _XOPEN_SOURCE 500
#include <stdlib.h>
int grantpt(int mfd);
                                             Returns 0 on success, or –1 on error
```

Thay đổi ownership và quyền của slave device tương ứng với master `mfd`. Trên Linux, pseudoterminal slave được tự động cấu hình đúng nên không cần gọi (nhưng portable application nên vẫn gọi).

### 64.2.3 Mở Khóa Slave: unlockpt()

```c
#define _XOPEN_SOURCE 500
#include <stdlib.h>
int unlockpt(int mfd);
                                             Returns 0 on success, or –1 on error
```

Xóa internal lock trên slave tương ứng với master `mfd`, cho phép process khác mở nó. Cố mở slave trước khi `unlockpt()` thất bại với `EIO`.

### 64.2.4 Lấy Tên Slave: ptsname()

```c
#define _XOPEN_SOURCE 500
#include <stdlib.h>
char *ptsname(int mfd);
               Returns pointer to (possibly statically allocated) string on success,
                                                                    or NULL on error
```

Trả về tên của pseudoterminal slave dạng `/dev/pts/nn`. Buffer thường được cấp phát tĩnh — bị ghi đè bởi lời gọi tiếp theo.

---

## 64.3 Mở Master: ptyMasterOpen()

Hàm `ptyMasterOpen()` đóng gói các bước mở pseudoterminal master và lấy tên slave:

```c
#include "pty_master_open.h"
int ptyMasterOpen(char *slaveName, size_t snLen);
                               Returns file descriptor on success, or –1 on error
```

**Listing 64-1:** Triển khai `ptyMasterOpen()`

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––– pty/pty_master_open.c
#define _XOPEN_SOURCE 600
#include <stdlib.h>
#include <fcntl.h>
#include "pty_master_open.h"
#include "tlpi_hdr.h"
int
ptyMasterOpen(char *slaveName, size_t snLen)
{
    int masterFd, savedErrno;
    char *p;
    masterFd = posix_openpt(O_RDWR | O_NOCTTY);
    if (masterFd == -1)  return -1;
    if (grantpt(masterFd) == -1) {
        savedErrno = errno;  close(masterFd);  errno = savedErrno;  return -1;
    }
    if (unlockpt(masterFd) == -1) {
        savedErrno = errno;  close(masterFd);  errno = savedErrno;  return -1;
    }
    p = ptsname(masterFd);
    if (p == NULL) {
        savedErrno = errno;  close(masterFd);  errno = savedErrno;  return -1;
    }
    if (strlen(p) < snLen) {
        strncpy(slaveName, p, snLen);
    } else {
        close(masterFd);  errno = EOVERFLOW;  return -1;
    }
    return masterFd;
}
––––––––––––––––––––––––––––––––––––––––––––––––––––– pty/pty_master_open.c
```

---

## 64.4 Kết Nối Process bằng Pseudoterminal: ptyFork()

Hàm `ptyFork()` tạo child process được kết nối với parent qua pseudoterminal pair:

```c
#include "pty_fork.h"
pid_t ptyFork(int *masterFd, char *slaveName, size_t snLen,
              const struct termios *slaveTermios, const struct winsize *slaveWS);
               In parent: returns process ID of child on success, or –1 on error;
                                  in successfully created child: always returns 0
```

Các bước trong `ptyFork()`:
1. Mở pseudoterminal master bằng `ptyMasterOpen()`.
2. Nếu `slaveName` không `NULL`, copy tên slave vào buffer.
3. Gọi `fork()`.
4. Parent trả về PID của child; master fd được trả về qua `*masterFd`.
5. Child:
   - `setsid()` để tạo session mới.
   - Đóng master fd.
   - Mở slave (trở thành controlling terminal).
   - Nếu `TIOCSCTTY` được định nghĩa, gọi `ioctl(slaveFd, TIOCSCTTY, 0)`.
   - Nếu `slaveTermios` không `NULL`, gọi `tcsetattr()` để đặt terminal attribute.
   - Nếu `slaveWS` không `NULL`, `ioctl(slaveFd, TIOCSWINSZ, ...)` để đặt window size.
   - `dup2()` để duplicate slave fd thành stdin/stdout/stderr.

---

## 64.5 Pseudoterminal I/O

Pseudoterminal pair tương tự pipe hai chiều: ghi vào master xuất hiện là input trên slave và ngược lại. Điểm khác biệt: **slave side hoạt động như terminal device**.

Ví dụ: ghi ký tự Control-C vào pseudoterminal master → slave tạo `SIGINT` cho foreground process group. Trong canonical mode, input được buffer theo dòng.

**Capacity**: Trên Linux, capacity là khoảng 4 kB mỗi chiều.

**Khi đóng master**: `SIGHUP` gửi đến controlling process của slave; `read()` trả về EOF; `write()` thất bại với `EIO`.

**Khi đóng slave**: `read()` từ master thất bại với `EIO`.

**Packet mode**: Cơ chế cho phép process trên pseudoterminal master được thông báo khi events liên quan đến software flow control xảy ra trên slave. Bật bằng `ioctl(mfd, TIOCPKT, &arg)`.

---

## 64.6 Triển Khai script(1)

`script(1)` đặt mình giữa terminal của user và shell, dùng pseudoterminal pair. Shell kết nối với pseudoterminal slave; `script` process kết nối với master. `script` relay data hai chiều và ghi tất cả output của master vào file `typescript`.

**Listing 64-3:** Implementation đơn giản của `script(1)`

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– pty/script.c
#include <sys/stat.h>
#include <fcntl.h>
#include <libgen.h>
#include <termios.h>
#include <sys/select.h>
#include "pty_fork.h"
#include "tty_functions.h"
#include "tlpi_hdr.h"
#define BUF_SIZE 256
#define MAX_SNAME 1000
struct termios ttyOrig;
static void
ttyReset(void)
{
    if (tcsetattr(STDIN_FILENO, TCSANOW, &ttyOrig) == -1)
        errExit("tcsetattr");
}
int
main(int argc, char *argv[])
{
    char slaveName[MAX_SNAME];
    char *shell;
    int masterFd, scriptFd;
    struct winsize ws;
    fd_set inFds;
    char buf[BUF_SIZE];
    ssize_t numRead;
    pid_t childPid;
    if (tcgetattr(STDIN_FILENO, &ttyOrig) == -1)  errExit("tcgetattr");
    if (ioctl(STDIN_FILENO, TIOCGWINSZ, &ws) < 0)  errExit("ioctl-TIOCGWINSZ");
    childPid = ptyFork(&masterFd, slaveName, MAX_SNAME, &ttyOrig, &ws);
    if (childPid == -1)  errExit("ptyFork");
    if (childPid == 0) {    /* Child: exec a shell on pty slave */
        shell = getenv("SHELL");
        if (shell == NULL || *shell == '\0')  shell = "/bin/sh";
        execlp(shell, shell, (char *) NULL);
        errExit("execlp");
    }
    /* Parent: relay data between terminal and pty master */
    scriptFd = open((argc > 1) ? argv[1] : "typescript",
                    O_WRONLY | O_CREAT | O_TRUNC,
                    S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP | S_IROTH | S_IWOTH);
    if (scriptFd == -1)  errExit("open typescript");
    ttySetRaw(STDIN_FILENO, &ttyOrig);
    if (atexit(ttyReset) != 0)  errExit("atexit");
    for (;;) {
        FD_ZERO(&inFds);
        FD_SET(STDIN_FILENO, &inFds);
        FD_SET(masterFd, &inFds);
        if (select(masterFd + 1, &inFds, NULL, NULL, NULL) == -1)  errExit("select");
        if (FD_ISSET(STDIN_FILENO, &inFds)) {    /* stdin --> pty */
            numRead = read(STDIN_FILENO, buf, BUF_SIZE);
            if (numRead <= 0)  exit(EXIT_SUCCESS);
            if (write(masterFd, buf, numRead) != numRead)
                fatal("partial/failed write (masterFd)");
        }
        if (FD_ISSET(masterFd, &inFds)) {    /* pty --> stdout+file */
            numRead = read(masterFd, buf, BUF_SIZE);
            if (numRead <= 0)  exit(EXIT_SUCCESS);
            if (write(STDOUT_FILENO, buf, numRead) != numRead)
                fatal("partial/failed write (STDOUT_FILENO)");
            if (write(scriptFd, buf, numRead) != numRead)
                fatal("partial/failed write (scriptFd)");
        }
    }
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– pty/script.c
```

---

## 64.7 Terminal Attribute và Window Size

Master và slave chia sẻ terminal attribute (`termios`) và window size (`winsize`). Chương trình trên pseudoterminal master có thể thay đổi các attribute này cho slave bằng cách áp dụng `tcsetattr()` và `ioctl()` lên file descriptor của master device.

Khi kích thước terminal emulator window thay đổi, cần:
1. Cài handler cho `SIGWINCH` trong parent `script` process.
2. Khi nhận `SIGWINCH`, lấy winsize mới và đặt cho pseudoterminal master qua `TIOCSWINSZ`.
3. Kernel tạo `SIGWINCH` cho foreground process group của pseudoterminal slave.

---

## 64.8 BSD Pseudoterminal

BSD pseudoterminal khác UNIX 98 chỉ ở cách tìm và mở cặp master/slave. Sau khi mở, hoạt động giống nhau.

BSD pseudoterminal là các entry được tạo trước trong thư mục `/dev`:
- Master: `/dev/ptyXY` (X ∈ `[p-za-e]`, Y ∈ `[0-9a-f]`)
- Slave: `/dev/ttyXY` (tương ứng)

Để tìm master chưa dùng, lặp qua danh sách master device cho đến khi `open()` thành công. Tên slave lấy bằng cách thay `pty` bằng `tty` trong tên master.

Dùng BSD pseudoterminal trên Linux không được khuyến nghị (từ Linux 2.6.4, hỗ trợ là optional qua `CONFIG_LEGACY_PTYS`).

---

## 64.9 Tóm Tắt

Pseudoterminal pair bao gồm master device và slave device được kết nối, cung cấp kênh IPC hai chiều. Lợi ích: slave behaves như terminal chuẩn, cho phép kết nối terminal-oriented program được điều khiển bởi driver program.

Ứng dụng phổ biến: network login service (ssh, telnet), terminal emulator, `script(1)`.

Hai API: System V (UNIX 98, được chuẩn hóa) và BSD. Linux hỗ trợ cả hai.

---

## 64.10 Bài Tập

- **64-1.** Thứ tự kết thúc của parent `script` process và child shell khi user gõ Control-D là gì?
- **64-2.** Thêm timestamp vào đầu/cuối file output; thêm code xử lý thay đổi window size.
- **64-3.** Sửa `script.c` để dùng pair of processes thay vì `select()`.
- **64-4.** Thêm time-stamped recording feature vào `script.c` và viết program `script_replay.c` để replay.
- **64-5.** Triển khai client và server cho simple telnet-style remote login.
- **64-6.** Thêm code cập nhật login accounting file cho chương trình bài 64-5.
- **64-7.** Viết chương trình `unbuffer` kết nối program với pseudoterminal để bypass block buffering của stdio.
