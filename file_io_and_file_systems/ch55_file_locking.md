## Chương 55
# Khóa File (File Locking)

Các chương trước đã đề cập đến các kỹ thuật mà process có thể dùng để đồng bộ hóa hành động, bao gồm signal (Chương 20–22) và semaphore (Chương 47 và 53). Trong chương này, chúng ta xem xét các kỹ thuật đồng bộ hóa bổ sung được thiết kế đặc biệt để dùng với file.

---

## 55.1 Tổng Quan

Một yêu cầu ứng dụng phổ biến là đọc dữ liệu từ file, thực hiện một số thay đổi, sau đó ghi lại vào file. Miễn là chỉ một process tại một thời điểm sử dụng file theo cách này thì không có vấn đề gì. Tuy nhiên, có thể xảy ra sự cố nếu nhiều process cùng lúc cập nhật file.

Ví dụ, nếu hai process cùng thực hiện các bước sau để cập nhật file chứa sequence number:
1. Đọc sequence number từ file.
2. Dùng sequence number cho mục đích ứng dụng.
3. Tăng sequence number và ghi lại vào file.

Vấn đề: kết quả cuối cùng file chứa giá trị 1001 trong khi đáng ra phải là 1002. Đây là race condition. Để ngăn chặn, ta cần đồng bộ hóa interprocess.

Mặc dù có thể dùng semaphore, dùng file lock thường ưa thích hơn vì kernel tự động liên kết lock với file.

Trong chương này, chúng ta mô tả hai API khác nhau để đặt file lock:
- `flock()`: đặt lock trên toàn bộ file (nguồn gốc từ BSD)
- `fcntl()`: đặt lock trên vùng của file (nguồn gốc từ System V)

Quy trình chung:
1. Đặt lock trên file.
2. Thực hiện file I/O.
3. Mở khóa file để process khác có thể lock.

### Kết hợp locking và hàm stdio

Vì stdio thực hiện buffering ở user-space, cần cẩn thận khi dùng hàm stdio với các kỹ thuật locking. Một số cách tránh vấn đề:
- Dùng `read()` và `write()` thay vì stdio.
- Flush stdio stream ngay sau khi đặt lock, và flush lại ngay trước khi giải phóng lock.
- Vô hiệu hóa stdio buffering bằng `setbuf()`.

### Advisory và mandatory locking

- **Advisory locking**: Mặc định. Process có thể bỏ qua lock được đặt bởi process khác.
- **Mandatory locking**: Buộc process thực hiện I/O phải tuân theo lock do process khác giữ.

---

## 55.2 File Locking với flock()

Mặc dù `fcntl()` cung cấp superset của chức năng của `flock()`, chúng ta vẫn mô tả `flock()` vì nó vẫn được dùng trong một số ứng dụng.

```c
#include <sys/file.h>
int flock(int fd, int operation);
                                             Returns 0 on success, or –1 on error
```

`flock()` đặt một lock duy nhất trên toàn bộ file. File cần lock được chỉ định qua file descriptor `fd`. Đối số `operation` chỉ định một trong các giá trị trong Bảng 55-1.

Theo mặc định, `flock()` block nếu process khác đang giữ lock không tương thích. Để ngăn điều này, OR `LOCK_NB` vào `operation`: nếu có lock không tương thích, `flock()` trả về `-1` với `errno` là `EWOULDBLOCK`.

**Bảng 55-1:** Giá trị cho đối số `operation` của `flock()`

| Giá trị | Mô tả |
|---------|-------|
| `LOCK_SH` | Đặt shared lock trên file được tham chiếu bởi `fd` |
| `LOCK_EX` | Đặt exclusive lock trên file được tham chiếu bởi `fd` |
| `LOCK_UN` | Mở khóa file được tham chiếu bởi `fd` |
| `LOCK_NB` | Thực hiện lock request không blocking |

Bất kỳ số process nào có thể đồng thời giữ shared lock trên file. Chỉ một process tại một thời điểm có thể giữ exclusive lock.

**Bảng 55-2:** Tương thích của các loại lock `flock()`

| Process A | Process B: `LOCK_SH` | Process B: `LOCK_EX` |
|-----------|----------------------|----------------------|
| `LOCK_SH` | Có | Không |
| `LOCK_EX` | Không | Không |

Process có thể đặt shared hoặc exclusive lock bất kể access mode (read, write, hoặc read-write) của file.

Có thể chuyển đổi shared lock sang exclusive lock (và ngược lại) bằng cách gọi `flock()` khác với giá trị `operation` thích hợp. Chuyển đổi shared lock sang exclusive lock sẽ block nếu process khác giữ shared lock, trừ khi `LOCK_NB` được chỉ định.

**Listing 55-1:** Sử dụng `flock()`

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––––– filelock/t_flock.c
#include <sys/file.h>
#include <fcntl.h>
#include "curr_time.h"
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    int fd, lock;
    const char *lname;
    if (argc < 3 || strcmp(argv[1], "--help") == 0 ||
        strchr("sx", argv[2][0]) == NULL)
        usageErr("%s file lock [sleep-time]\n"
                 "  'lock' is 's' (shared) or 'x' (exclusive)\n"
                 "  optionally followed by 'n' (nonblocking)\n"
                 "  'secs' specifies time to hold lock\n", argv[0]);
    lock = (argv[2][0] == 's') ? LOCK_SH : LOCK_EX;
    if (argv[2][1] == 'n')
        lock |= LOCK_NB;
    fd = open(argv[1], O_RDONLY);
    if (fd == -1)  errExit("open");
    lname = (lock & LOCK_SH) ? "LOCK_SH" : "LOCK_EX";
    printf("PID %ld: requesting %s at %s\n", (long) getpid(), lname,
           currTime("%T"));
    if (flock(fd, lock) == -1) {
        if (errno == EWOULDBLOCK)
            fatal("PID %ld: already locked - bye!", (long) getpid());
        else
            errExit("flock (PID=%ld)", (long) getpid());
    }
    printf("PID %ld: granted %s at %s\n", (long) getpid(), lname,
           currTime("%T"));
    sleep((argc > 3) ? getInt(argv[3], GN_NONNEG, "sleep-time") : 10);
    printf("PID %ld: releasing %s at %s\n", (long) getpid(), lname,
           currTime("%T"));
    if (flock(fd, LOCK_UN) == -1)  errExit("flock");
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– filelock/t_flock.c
```

### 55.2.1 Ngữ Nghĩa của Kế Thừa và Giải Phóng Lock

Lock `flock()` được liên kết với **open file description** (Mục 5.4), không phải file descriptor hay bản thân file (i-node). Điều này có nghĩa:

- Khi file descriptor được duplicate (`dup()`, `dup2()`, `fcntl() F_DUPFD`), descriptor mới tham chiếu đến cùng lock.
- Lock được giải phóng chỉ khi tất cả duplicate descriptor đã được đóng (hoặc khi thực hiện unlock rõ ràng).
- Nếu dùng `open()` để mở lại cùng file (tạo open file description mới), descriptor thứ hai được xử lý độc lập — process có thể lock chính mình.
- Khi tạo child process bằng `fork()`, child nhận bản sao descriptor của parent và tham chiếu đến cùng lock.
- Lock được bảo toàn qua `exec()` (trừ khi close-on-exec flag được đặt).

### 55.2.2 Hạn Chế của flock()

- Chỉ có thể lock toàn bộ file (không thể lock vùng cụ thể).
- Chỉ có thể đặt advisory lock.
- Nhiều triển khai NFS không nhận biết lock của `flock()`.

---

## 55.3 Record Locking với fcntl()

Dùng `fcntl()` (Mục 5.2), ta có thể đặt lock trên bất kỳ phần nào của file, từ một byte đến toàn bộ file. Dạng locking này thường gọi là **record locking** (dù thực ra là byte range locking). Đây là dạng locking duy nhất được quy định trong POSIX.1 gốc và SUSv3.

Dạng chung của lời gọi `fcntl()` để tạo hoặc xóa file lock:

```c
struct flock flockstr;
/* Đặt các trường của 'flockstr' để mô tả lock cần đặt hoặc xóa */
fcntl(fd, cmd, &flockstr);
```

### Structure flock

Structure `flock` định nghĩa lock cần acquire hoặc xóa:

```c
struct flock {
    short l_type;    /* Loại lock: F_RDLCK, F_WRLCK, F_UNLCK */
    short l_whence;  /* Cách diễn giải 'l_start': SEEK_SET, SEEK_CUR, SEEK_END */
    off_t l_start;   /* Offset bắt đầu lock */
    off_t l_len;     /* Số byte cần lock; 0 = "đến EOF" */
    pid_t l_pid;     /* Process ngăn lock của ta (chỉ F_GETLK) */
};
```

**Bảng 55-3:** Loại lock cho fcntl() locking

| Loại lock | Mô tả |
|-----------|-------|
| `F_RDLCK` | Đặt read lock |
| `F_WRLCK` | Đặt write lock |
| `F_UNLCK` | Xóa lock hiện có |

Read lock (F_RDLCK) và write lock (F_WRLCK) tương ứng với shared và exclusive lock của `flock()`:
- Bất kỳ số process nào có thể giữ read lock trên một vùng.
- Chỉ một process có thể giữ write lock, và lock đó loại trừ read và write lock của process khác.

Để đặt read lock cần file được mở để đọc; để đặt write lock cần mở để ghi.

Các trường `l_whence`, `l_start`, và `l_len` xác định phạm vi byte cần lock (tương tự `lseek()`). Chỉ định `l_len = 0` có nghĩa "lock tất cả byte từ điểm chỉ định đến EOF". Để lock toàn bộ file: `l_whence = SEEK_SET`, `l_start = l_len = 0`.

### Đối số cmd

- **`F_SETLK`**: Acquire (`l_type` là `F_RDLCK` hoặc `F_WRLCK`) hoặc release (`l_type` là `F_UNLCK`) lock. Nếu lock không tương thích tồn tại, thất bại với `EAGAIN` (hoặc `EACCES`).
- **`F_SETLKW`**: Giống `F_SETLK`, nhưng block nếu có lock không tương thích. Có thể bị interrupt bởi signal (lỗi `EINTR`).
- **`F_GETLK`**: Kiểm tra xem có thể đặt lock không, nhưng không thực sự đặt. Trả về thông tin về lock hiện có nếu có conflict.

**Lưu ý về F_GETLK**: Có race condition tiềm ẩn khi kết hợp `F_GETLK` với `F_SETLK` tiếp theo. Phải chuẩn bị cho khả năng lỗi từ `F_SETLK` ngay cả khi `F_GETLK` nói rằng có thể đặt lock.

### Chi tiết về acquire và release lock

- Mở khóa vùng file luôn thành công ngay lập tức.
- Process có thể giữ chỉ một loại lock trên một vùng cụ thể. Đặt lock mới trên vùng đã lock sẽ chuyển đổi atomically sang mode mới.
- Process không bao giờ có thể lock chính mình ra khỏi vùng file, dù qua nhiều file descriptor (khác với `flock()`).

### 55.3.1 Deadlock

Khi dùng `F_SETLKW`, cần biết về tình huống deadlock. Kernel kiểm tra mỗi lock request mới qua `F_SETLKW` để xem có dẫn đến deadlock không. Nếu có, kernel chọn một trong các process bị block và làm lời gọi `fcntl()` của nó thất bại với `EDEADLK`.

Deadlock được phát hiện ngay cả khi đặt lock trên nhiều file khác nhau, và circular deadlock liên quan nhiều process.

### 55.3.2 Ví dụ: Chương Trình Locking Tương Tác

Listing 55-2 cho phép thử nghiệm record locking tương tác.

**Listing 55-2:** Chương trình fcntl() locking tương tác

```c
–––––––––––––––––––––––––––––––––––––––––––– filelock/i_fcntl_locking.c
#include <sys/stat.h>
#include <fcntl.h>
#include "tlpi_hdr.h"
#define MAX_LINE 100
int
main(int argc, char *argv[])
{
    int fd, numRead, cmd, status;
    char lock, cmdCh, whence, line[MAX_LINE];
    struct flock fl;
    long long len, st;
    if (argc != 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s file\n", argv[0]);
    fd = open(argv[1], O_RDWR);
    if (fd == -1)  errExit("open (%s)", argv[1]);
    printf("Enter ? for help\n");
    for (;;) {
        printf("PID=%ld> ", (long) getpid());
        fflush(stdout);
        if (fgets(line, MAX_LINE, stdin) == NULL)  exit(EXIT_SUCCESS);
        line[strlen(line) - 1] = '\0';
        if (*line == '\0')  continue;
        whence = 's';
        numRead = sscanf(line, "%c %c %lld %lld %c",
                         &cmdCh, &lock, &st, &len, &whence);
        fl.l_start = st;
        fl.l_len = len;
        if (numRead < 4 || strchr("gsw", cmdCh) == NULL ||
            strchr("rwu", lock) == NULL || strchr("sce", whence) == NULL) {
            printf("Invalid command!\n");
            continue;
        }
        cmd = (cmdCh == 'g') ? F_GETLK :
              (cmdCh == 's') ? F_SETLK : F_SETLKW;
        fl.l_type   = (lock == 'r') ? F_RDLCK :
                      (lock == 'w') ? F_WRLCK : F_UNLCK;
        fl.l_whence = (whence == 'c') ? SEEK_CUR :
                      (whence == 'e') ? SEEK_END : SEEK_SET;
        status = fcntl(fd, cmd, &fl);
        if (cmd == F_GETLK) {
            if (status == -1) {
                errMsg("fcntl - F_GETLK");
            } else {
                if (fl.l_type == F_UNLCK)
                    printf("[PID=%ld] Lock can be placed\n", (long) getpid());
                else
                    printf("[PID=%ld] Denied by %s lock on %lld:%lld "
                           "(held by PID %ld)\n", (long) getpid(),
                           (fl.l_type == F_RDLCK) ? "READ" : "WRITE",
                           (long long) fl.l_start,
                           (long long) fl.l_len, (long) fl.l_pid);
            }
        } else {
            if (status == 0)
                printf("[PID=%ld] %s\n", (long) getpid(),
                       (lock == 'u') ? "unlocked" : "got lock");
            else if (errno == EAGAIN || errno == EACCES)
                printf("[PID=%ld] failed (incompatible lock)\n", (long) getpid());
            else if (errno == EDEADLK)
                printf("[PID=%ld] failed (deadlock)\n", (long) getpid());
            else
                errMsg("fcntl - F_SETLK(W)");
        }
    }
}
–––––––––––––––––––––––––––––––––––––––––––– filelock/i_fcntl_locking.c
```

### 55.3.3 Ví dụ: Thư Viện Các Hàm Locking

Listing 55-3 cung cấp tập hàm locking có thể dùng trong các chương trình khác:
- `lockRegion()`: dùng `F_SETLK` để đặt lock.
- `lockRegionWait()`: dùng `F_SETLKW`, blocking lock request.
- `regionIsLocked()`: kiểm tra xem lock có thể đặt không.

**Listing 55-3:** Hàm locking vùng file

```c
––––––––––––––––––––––––––––––––––––––––––– filelock/region_locking.c
#include <fcntl.h>
#include "region_locking.h"
static int
lockReg(int fd, int cmd, int type, int whence, int start, off_t len)
{
    struct flock fl;
    fl.l_type   = type;
    fl.l_whence = whence;
    fl.l_start  = start;
    fl.l_len    = len;
    return fcntl(fd, cmd, &fl);
}
int
lockRegion(int fd, int type, int whence, int start, int len)
{
    return lockReg(fd, F_SETLK, type, whence, start, len);
}
int
lockRegionWait(int fd, int type, int whence, int start, int len)
{
    return lockReg(fd, F_SETLKW, type, whence, start, len);
}
pid_t
regionIsLocked(int fd, int type, int whence, int start, int len)
{
    struct flock fl;
    fl.l_type   = type;
    fl.l_whence = whence;
    fl.l_start  = start;
    fl.l_len    = len;
    if (fcntl(fd, F_GETLK, &fl) == -1)
        return -1;
    return (fl.l_type == F_UNLCK) ? 0 : fl.l_pid;
}
––––––––––––––––––––––––––––––––––––––––––– filelock/region_locking.c
```

### 55.3.4 Giới Hạn Lock và Hiệu Năng

Linux không đặt giới hạn trên cố định về số record lock có thể acquire; chỉ bị giới hạn bởi tính khả dụng của memory.

Mỗi open file có danh sách liên kết các lock liên quan, được sắp xếp theo process ID rồi theo starting offset. Thời gian cần để thêm hoặc xóa lock tăng gần tuyến tính với số lock hiện có trên file.

### 55.3.5 Ngữ Nghĩa của Kế Thừa và Giải Phóng Lock

Ngữ nghĩa của fcntl() record lock khác đáng kể với `flock()`:

- **Record lock không được kế thừa qua `fork()`** — khác với `flock()` nơi child kế thừa tham chiếu đến cùng lock.
- **Record lock được bảo toàn qua `exec()`**.
- **Tất cả thread trong process chia sẻ cùng tập record lock**.
- **Record lock liên kết với cả process lẫn i-node**. Khi process kết thúc, tất cả record lock được giải phóng. Nghiêm trọng hơn: khi process **đóng bất kỳ file descriptor nào**, tất cả lock mà process giữ trên file tương ứng được giải phóng, bất kể lock được acquire qua file descriptor nào.

```c
/* Ví dụ: close(fd2) giải phóng lock được acquire qua fd1 */
fd1 = open("testfile", O_RDWR);
fd2 = open("testfile", O_RDWR);
if (fcntl(fd1, F_SETLKW, &fl) == -1) errExit("fcntl");
close(fd2);  /* Giải phóng lock trên testfile dù lock qua fd1! */
```

Ngữ nghĩa này là blemish kiến trúc, làm cho việc dùng record lock từ library package trở nên phức tạp.

### 55.3.6 Starvation Lock và Ưu Tiên của Queued Lock Request

Trên Linux:
- Thứ tự các queued lock request được cấp là không xác định.
- Writer không có ưu tiên hơn reader và ngược lại.
- Một loạt read lock có thể làm blocked write lock bị starvation.

---

## 55.4 Mandatory Locking

Các loại lock mô tả đến nay là advisory — process có thể bỏ qua lock của process khác. Linux cũng cho phép fcntl() record lock là mandatory, nghĩa là mọi thao tác file I/O được kiểm tra xem có tương thích với lock nào mà process khác đang giữ không.

Để dùng mandatory locking, phải bật trên file system và từng file cụ thể:

```
# mount -o mand /dev/sda10 /testfs
```

Bật mandatory locking trên file bằng tổ hợp: bit set-group-ID bật VÀ quyền group-execute tắt:

```
$ chmod g+s,g-x /testfs/file
$ ls -l /testfs/file
-rw-r-Sr-- 1 mtk users 0 Apr 22 14:11 /testfs/file
```

### Ảnh hưởng của mandatory locking lên file I/O

Khi system call thực hiện data transfer gặp lock conflict:
- Blocking mode: system call block.
- Non-blocking mode (`O_NONBLOCK`): thất bại với `EAGAIN`.

Deadlock tương tự có thể xảy ra với mandatory lock — kernel giải quyết bằng cách làm `write()` của một process thất bại với `EDEADLK`.

### Cảnh báo về mandatory locking

- Giữ mandatory lock không ngăn process khác xóa file (chỉ cần quyền thích hợp trên parent directory).
- Tạo rủi ro denial-of-service nếu file có thể truy cập công khai.
- Chi phí hiệu năng cho mỗi I/O system call trên file có mandatory locking.
- Phức tạp trong thiết kế ứng dụng — phải xử lý `EAGAIN` và `EDEADLK`.
- Có race condition trong triển khai Linux hiện tại.

**Tóm lại: nên tránh dùng mandatory lock.**

---

## 55.5 File /proc/locks

Có thể xem tập lock hiện đang được giữ trong hệ thống bằng cách kiểm tra nội dung file `/proc/locks`:

```
$ cat /proc/locks
1: POSIX ADVISORY WRITE 458 03:07:133880 0 EOF
2: FLOCK ADVISORY WRITE 404 03:07:133875 0 EOF
3: POSIX ADVISORY WRITE 312 03:07:133853 0 EOF
4: FLOCK ADVISORY WRITE 274 03:07:81908 0 EOF
```

Tám trường hiển thị cho mỗi lock (từ trái sang phải):
1. Số thứ tự của lock.
2. Loại lock: `FLOCK` (tạo bởi `flock()`) hoặc `POSIX` (tạo bởi `fcntl()`).
3. Mode của lock: `ADVISORY` hoặc `MANDATORY`.
4. Loại lock: `READ` hoặc `WRITE`.
5. Process ID của process giữ lock.
6. Ba số phân cách bởi dấu hai chấm: major và minor device number của file system, theo sau là i-node number của file.
7. Starting byte của lock.
8. Ending byte của lock (`EOF` = `l_len` là 0).

Các dòng có ký tự `->` sau số lock đại diện cho lock request bị block bởi lock tương ứng.

---

## 55.6 Chạy Chỉ Một Instance của Chương Trình

Một số chương trình (đặc biệt là daemon) cần đảm bảo chỉ một instance đang chạy. Phương pháp phổ biến:

1. Daemon tạo file trong thư mục chuẩn (thường `/var/run`) và đặt write lock.
2. Daemon giữ lock trong suốt thời gian thực thi và xóa file trước khi kết thúc.
3. Nếu instance khác được khởi động, nó sẽ thất bại khi cố acquire write lock và biết rằng instance khác đang chạy.

Theo quy ước, daemon ghi process ID vào file lock (thường có đuôi `.pid`, ví dụ `/var/run/syslogd.pid`).

**Listing 55-4:** Tạo PID lock file để đảm bảo chỉ một instance chương trình

```c
–––––––––––––––––––––––––––––––––––––––––––– filelock/create_pid_file.c
#include <sys/stat.h>
#include <fcntl.h>
#include "region_locking.h"
#include "create_pid_file.h"
#include "tlpi_hdr.h"
#define BUF_SIZE 100
int
createPidFile(const char *progName, const char *pidFile, int flags)
{
    int fd;
    char buf[BUF_SIZE];
    fd = open(pidFile, O_RDWR | O_CREAT, S_IRUSR | S_IWUSR);
    if (fd == -1)
        errExit("Could not open PID file %s", pidFile);
    if (flags & CPF_CLOEXEC) {
        flags = fcntl(fd, F_GETFD);
        if (flags == -1)
            errExit("Could not get flags for PID file %s", pidFile);
        flags |= FD_CLOEXEC;
        if (fcntl(fd, F_SETFD, flags) == -1)
            errExit("Could not set flags for PID file %s", pidFile);
    }
    if (lockRegion(fd, F_WRLCK, SEEK_SET, 0, 0) == -1) {
        if (errno == EAGAIN || errno == EACCES)
            fatal("PID file '%s' is locked; probably "
                  "'%s' is already running", pidFile, progName);
        else
            errExit("Unable to lock PID file '%s'", pidFile);
    }
    if (ftruncate(fd, 0) == -1)
        errExit("Could not truncate PID file '%s'", pidFile);
    snprintf(buf, BUF_SIZE, "%ld\n", (long) getpid());
    if (write(fd, buf, strlen(buf)) != strlen(buf))
        fatal("Writing to PID file '%s'", pidFile);
    return fd;
}
–––––––––––––––––––––––––––––––––––––––––––– filelock/create_pid_file.c
```

Dùng `ftruncate()` để xóa bất kỳ chuỗi trước đó trong lock file — cần thiết vì instance cuối có thể không xóa file (ví dụ: do crash hệ thống).

---

## 55.7 Kỹ Thuật Locking Cũ

Trong các triển khai UNIX cũ thiếu file locking, nhiều kỹ thuật ad hoc được dùng. Tất cả đều là advisory.

### open(file, O_CREAT | O_EXCL,...) + unlink(file)

Tận dụng tính atomic của `open()` với `O_CREAT|O_EXCL`. Hạn chế: phải polling để retry, chậm hơn record lock, không tự động giải phóng khi process crash, không phát hiện deadlock.

### link(file, lockfile) + unlink(lockfile)

Tận dụng thực tế `link()` thất bại nếu link mới đã tồn tại. Cùng hạn chế như trên.

### open(file, O_CREAT | O_TRUNC | O_WRONLY, 0) + unlink(file)

Tận dụng thực tế `open()` với `O_TRUNC` thất bại nếu quyền ghi bị từ chối. Không thể dùng với chương trình có đặc quyền superuser.

---

## 55.8 Tóm Tắt

File lock cho phép các process đồng bộ hóa truy cập vào file. Linux cung cấp hai system call file locking: `flock()` (nguồn gốc BSD) và `fcntl()` (nguồn gốc System V). Chỉ `fcntl()` locking được chuẩn hóa trong SUSv3.

`flock()` lock toàn bộ file: shared lock (nhiều process cùng giữ được) và exclusive lock (ngăn mọi lock khác). `fcntl()` đặt "record lock" trên bất kỳ vùng nào, từ một byte đến toàn bộ file. Nếu blocking lock request (`F_SETLKW`) sẽ gây deadlock, kernel làm `fcntl()` thất bại với `EDEADLK`.

Lock của `flock()` và `fcntl()` vô hình với nhau. Hai loại có ngữ nghĩa khác nhau về kế thừa qua `fork()` và giải phóng khi đóng file descriptor.

File `/proc/locks` hiển thị tất cả file lock hiện đang được giữ trên hệ thống.

---

## 55.9 Bài Tập

- **55-1.** Thử nghiệm với chương trình trong Listing 55-1 để xác định: (a) Shared lock có thể làm exclusive lock bị starvation không? (b) Quy tắc nào xác định process nào được cấp lock tiếp theo?
- **55-2.** Viết chương trình kiểm tra xem `flock()` có phát hiện deadlock khi lock hai file khác nhau trong hai process không.
- **55-3.** Viết chương trình xác nhận các nhận định trong Mục 55.2.1 về ngữ nghĩa kế thừa và giải phóng lock `flock()`.
- **55-4.** Thử nghiệm để xem lock của `flock()` và `fcntl()` có ảnh hưởng lẫn nhau không.
- **55-5.** Viết hai chương trình để xác nhận rằng thời gian thêm hoặc kiểm tra sự tồn tại của lock tăng tuyến tính với số lock hiện có trên file.
- **55-6.** Thử nghiệm để xác nhận các nhận định trong Mục 55.3.6 về starvation lock.
- **55-8.** Dùng chương trình trong Listing 55-2 để chứng minh kernel phát hiện circular deadlock liên quan ba (hoặc nhiều hơn) process.
- **55-9.** Viết cặp chương trình để tạo ra tình huống deadlock với mandatory lock mô tả trong Mục 55.4.
- **55-10.** Viết phiên bản đơn giản của tiện ích `lockfile(1)`.
