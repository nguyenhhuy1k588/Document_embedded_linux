## Chương 15
# Thuộc Tính File (File Attributes)

Trong chương này, chúng ta khảo sát các thuộc tính khác nhau của file (metadata). Chúng ta bắt đầu với mô tả về system call `stat()`, trả về một structure chứa nhiều thuộc tính này, gồm file timestamp, quyền sở hữu file, và quyền truy cập file. Sau đó chúng ta xem xét các system call dùng để thay đổi các thuộc tính này. Kết thúc chương là thảo luận về i-node flag (còn gọi là ext2 extended file attribute), kiểm soát các khía cạnh khác nhau trong cách kernel xử lý file.

---

## 15.1 Lấy Thông Tin File: stat()

Các system call `stat()`, `lstat()`, và `fstat()` lấy thông tin về file, chủ yếu từ i-node của file.

```c
#include <sys/stat.h>
int stat(const char *pathname, struct stat *statbuf);
int lstat(const char *pathname, struct stat *statbuf);
int fstat(int fd, struct stat *statbuf);
                                           All return 0 on success, or –1 on error
```

Ba system call này chỉ khác nhau ở cách xác định file:

- `stat()` trả về thông tin về file có tên đã chỉ định;
- `lstat()` tương tự `stat()`, ngoại trừ nếu file được đặt tên là symbolic link, thông tin về chính link đó được trả về, thay vì file mà link trỏ đến; và
- `fstat()` trả về thông tin về file được tham chiếu bởi file descriptor đã mở.

Các system call `stat()` và `lstat()` không yêu cầu quyền trên chính file. Tuy nhiên, cần quyền execute (search) trên tất cả các parent directory được chỉ định trong pathname. System call `fstat()` luôn thành công nếu được cung cấp file descriptor hợp lệ.

Tất cả các system call này trả về structure `stat` trong buffer được trỏ bởi `statbuf`. Structure này có dạng sau:

```c
struct stat {
    dev_t     st_dev;     /* IDs of device on which file resides */
    ino_t     st_ino;     /* I-node number of file */
    mode_t    st_mode;    /* File type and permissions */
    nlink_t   st_nlink;   /* Number of (hard) links to file */
    uid_t     st_uid;     /* User ID of file owner */
    gid_t     st_gid;     /* Group ID of file owner */
    dev_t     st_rdev;    /* IDs for device special files */
    off_t     st_size;    /* Total file size (bytes) */
    blksize_t st_blksize; /* Optimal block size for I/O (bytes) */
    blkcnt_t  st_blocks;  /* Number of (512B) blocks allocated */
    time_t    st_atime;   /* Time of last file access */
    time_t    st_mtime;   /* Time of last file modification */
    time_t    st_ctime;   /* Time of last status change */
};
```

### Device ID và i-node number

Trường `st_dev` xác định thiết bị chứa file. Trường `st_ino` chứa i-node number của file. Tổ hợp `st_dev` và `st_ino` xác định duy nhất một file trên tất cả các file system.

Nếu đây là i-node cho device, thì trường `st_rdev` chứa major và minor ID của thiết bị.

Major và minor ID của giá trị `dev_t` có thể được trích xuất bằng hai macro: `major()` và `minor()`.

### Quyền sở hữu file

Các trường `st_uid` và `st_gid` xác định lần lượt owner (user ID) và group (group ID) mà file thuộc về.

### Link count

Trường `st_nlink` là số hard link đến file. Chúng ta mô tả link chi tiết trong Chương 18.

### Loại file và quyền truy cập

Trường `st_mode` là bit mask phục vụ mục đích kép: xác định loại file và chỉ định quyền truy cập file. Bố cục của trường này như sau:

```
←─ Loại file ─→ ←────────────── Quyền ────────────────→
┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐
│   │   │   │   │ U │ G │ T │ R │ W │ X │ R │ W │ X │ R │ W │ X │
└───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘
                                └───────┘ └───────┘ └───────┘
                                  User      Group      Other
```

Loại file có thể được trích xuất từ trường này bằng cách AND (`&`) với hằng số `S_IFMT`. Kết quả sau đó có thể được so sánh với các hằng số để xác định loại file:

```c
if ((statbuf.st_mode & S_IFMT) == S_IFREG)
    printf("regular file\n");
```

Vì đây là thao tác phổ biến, các macro chuẩn được cung cấp để đơn giản hóa:

```c
if (S_ISREG(statbuf.st_mode))
    printf("regular file\n");
```

**Bảng 15-1:** Macro kiểm tra loại file trong trường `st_mode`

| Hằng số | Macro kiểm tra | Loại file |
|---------|----------------|-----------|
| `S_IFREG` | `S_ISREG()` | File thông thường |
| `S_IFDIR` | `S_ISDIR()` | Directory |
| `S_IFCHR` | `S_ISCHR()` | Character device |
| `S_IFBLK` | `S_ISBLK()` | Block device |
| `S_IFIFO` | `S_ISFIFO()` | FIFO hoặc pipe |
| `S_IFSOCK` | `S_ISSOCK()` | Socket |
| `S_IFLNK` | `S_ISLNK()` | Symbolic link |

### Kích thước file, block được cấp phát, và kích thước block I/O tối ưu

Với file thông thường, trường `st_size` là tổng kích thước file tính bằng byte. Với symbolic link, trường này chứa độ dài (byte) của pathname được link trỏ đến.

Trường `st_blocks` chỉ ra tổng số block được cấp phát cho file, tính bằng đơn vị block 512 byte. Nếu file chứa hole (Mục 4.7), giá trị này sẽ nhỏ hơn mong đợi so với kích thước tương ứng tính bằng byte.

Trường `st_blksize` là kích thước block tối ưu (byte) cho I/O trên file trong file system này. I/O với block nhỏ hơn kích thước này kém hiệu quả hơn. Giá trị điển hình là 4096.

### File timestamp

Các trường `st_atime`, `st_mtime`, và `st_ctime` chứa lần lượt thời gian truy cập file lần cuối, thời gian sửa đổi file lần cuối, và thời gian thay đổi trạng thái lần cuối. Các trường này có kiểu `time_t` — số giây kể từ Epoch.

### Chương trình ví dụ

Chương trình trong Listing 15-1 dùng `stat()` để lấy thông tin về file được đặt tên trên command line.

```
$ echo 'All operating systems provide services for programs they run' > apue
$ chmod g+s apue
$ ./t_stat apue
File type:                  regular file
Device containing i-node:   major=3 minor=11
I-node number:              234363
Mode:                       102644 (rw-r--r--)
  special bits set:         set-GID
Number of (hard) links:     1
Ownership:                  UID=1000 GID=100
File size:                  61 bytes
Optimal I/O block size:     4096 bytes
512B blocks allocated:      8
Last file access:           Mon Jun  8 09:40:07 2011
Last file modification:     Mon Jun  8 09:39:25 2011
Last status change:         Mon Jun  8 09:39:51 2011
```

**Listing 15-1:** Lấy và diễn giải thông tin `stat` của file

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– files/t_stat.c
#define _BSD_SOURCE /* Get major() and minor() from <sys/types.h> */
#include <sys/types.h>
#include <sys/stat.h>
#include <time.h>
#include "file_perms.h"
#include "tlpi_hdr.h"
static void
displayStatInfo(const struct stat *sb)
{
    printf("File type:                  ");
    switch (sb->st_mode & S_IFMT) {
    case S_IFREG:  printf("regular file\n");           break;
    case S_IFDIR:  printf("directory\n");              break;
    case S_IFCHR:  printf("character device\n");       break;
    case S_IFBLK:  printf("block device\n");           break;
    case S_IFLNK:  printf("symbolic (soft) link\n");   break;
    case S_IFIFO:  printf("FIFO or pipe\n");           break;
    case S_IFSOCK: printf("socket\n");                 break;
    default:       printf("unknown file type?\n");     break;
    }
    printf("Device containing i-node:   major=%ld minor=%ld\n",
           (long) major(sb->st_dev), (long) minor(sb->st_dev));
    printf("I-node number:              %ld\n", (long) sb->st_ino);
    printf("Mode:                       %lo (%s)\n",
           (unsigned long) sb->st_mode, filePermStr(sb->st_mode, 0));
    if (sb->st_mode & (S_ISUID | S_ISGID | S_ISVTX))
        printf("  special bits set:         %s%s%s\n",
               (sb->st_mode & S_ISUID) ? "set-UID " : "",
               (sb->st_mode & S_ISGID) ? "set-GID " : "",
               (sb->st_mode & S_ISVTX) ? "sticky "  : "");
    printf("Number of (hard) links:     %ld\n", (long) sb->st_nlink);
    printf("Ownership:                  UID=%ld GID=%ld\n",
           (long) sb->st_uid, (long) sb->st_gid);
    if (S_ISCHR(sb->st_mode) || S_ISBLK(sb->st_mode))
        printf("Device number (st_rdev):    major=%ld; minor=%ld\n",
               (long) major(sb->st_rdev), (long) minor(sb->st_rdev));
    printf("File size:                  %lld bytes\n",  (long long) sb->st_size);
    printf("Optimal I/O block size:     %ld bytes\n",   (long) sb->st_blksize);
    printf("512B blocks allocated:      %lld\n",        (long long) sb->st_blocks);
    printf("Last file access:           %s", ctime(&sb->st_atime));
    printf("Last file modification:     %s", ctime(&sb->st_mtime));
    printf("Last status change:         %s", ctime(&sb->st_ctime));
}
int
main(int argc, char *argv[])
{
    struct stat sb;
    Boolean statLink;
    int fname;
    statLink = (argc > 1) && strcmp(argv[1], "-l") == 0;
    fname = statLink ? 2 : 1;
    if (fname >= argc || (argc > 1 && strcmp(argv[1], "--help") == 0))
        usageErr("%s [-l] file\n"
                 "  -l = use lstat() instead of stat()\n", argv[0]);
    if (statLink) {
        if (lstat(argv[fname], &sb) == -1) errExit("lstat");
    } else {
        if (stat(argv[fname], &sb) == -1)  errExit("stat");
    }
    displayStatInfo(&sb);
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– files/t_stat.c
```

---

## 15.2 File Timestamp

Các trường `st_atime`, `st_mtime`, và `st_ctime` của structure `stat` chứa file timestamp. Chúng ghi lại lần lượt thời gian truy cập file lần cuối, thời gian sửa đổi file lần cuối, và thời gian thay đổi trạng thái file lần cuối (tức là lần thay đổi thông tin i-node lần cuối). Timestamp được ghi bằng số giây kể từ Epoch.

**Bảng 15-2:** Ảnh hưởng của các hàm khác nhau lên file timestamp

| Hàm | Timestamp file/dir thay đổi | Ghi chú |
|-----|---------------------------|---------|
| `chmod()` | c | Tương tự `fchmod()` |
| `chown()` | c | Tương tự `lchown()` và `fchown()` |
| `exec()` | a | |
| `link()` | c trên file; m,c trên parent dir | |
| `mkdir()` | a,m,c; m,c trên parent dir | |
| `open()`, `creat()` | a,m,c; m,c trên parent dir | Khi tạo file mới |
| `open()`, `creat()` | m,c | Khi truncate file hiện có |
| `read()` | a | Tương tự `readv()`, `pread()` |
| `rename()` | c trên file; m,c trên cả hai parent dir | |
| `rmdir()` | m,c trên parent dir | |
| `truncate()` | m,c | Chỉ khi kích thước file thay đổi |
| `unlink()` | c trên file; m,c trên parent dir | |
| `utime()` | a,m,c | Tương tự `utimes()`, `futimes()`, `utimensat()` |
| `write()` | m,c | Tương tự `writev()`, `pwrite()` |

### Nanosecond timestamp

Với phiên bản 2.6, Linux hỗ trợ độ phân giải nanosecond cho ba trường timestamp của structure `stat`. Không phải tất cả file system đều hỗ trợ timestamp nanosecond. JFS, XFS, ext4, và Btrfs hỗ trợ, nhưng ext2, ext3, và Reiserfs thì không.

Dưới glibc API (từ phiên bản 2.3), các trường timestamp được định nghĩa là structure `timespec`. Thành phần nanosecond có thể được truy cập bằng tên trường như `st_atim.tv_nsec`.

### 15.2.1 Thay Đổi File Timestamp bằng utime() và utimes()

Last file access và modification timestamp được lưu trong i-node có thể thay đổi rõ ràng bằng `utime()` hoặc một trong các system call liên quan. Các chương trình như `tar(1)` và `unzip(1)` dùng các system call này để reset file timestamp khi giải nén archive.

```c
#include <utime.h>
int utime(const char *pathname, const struct utimbuf *buf);
                                            Returns 0 on success, or –1 on error
```

Đối số `buf` có thể là `NULL` hoặc pointer đến structure `utimbuf`:

```c
struct utimbuf {
    time_t actime;   /* Access time */
    time_t modtime;  /* Modification time */
};
```

Hai trường hợp khác nhau xác định cách `utime()` hoạt động:

- Nếu `buf` là `NULL`: cả last access lẫn last modification time được đặt thành thời gian hiện tại. Effective user ID của process phải khớp với user ID của file, hoặc process phải có quyền ghi trên file, hoặc process phải có đặc quyền (`CAP_FOWNER` hoặc `CAP_DAC_OVERRIDE`).
- Nếu `buf` là pointer đến structure `utimbuf`: last file access và modification time được cập nhật bằng các trường tương ứng. Effective user ID của process phải khớp với user ID của file (chỉ có quyền ghi là không đủ), hoặc process phải có đặc quyền (`CAP_FOWNER`).

Ví dụ — làm cho last modification time của file bằng last access time:

```c
struct stat sb;
struct utimbuf utb;
if (stat(pathname, &sb) == -1)
    errExit("stat");
utb.actime  = sb.st_atime; /* Giữ nguyên access time */
utb.modtime = sb.st_atime;
if (utime(pathname, &utb) == -1)
    errExit("utime");
```

Linux cũng cung cấp `utimes()` (dẫn xuất BSD), cho phép chỉ định giá trị thời gian với độ chính xác microsecond:

```c
#include <sys/time.h>
int utimes(const char *pathname, const struct timeval tv[2]);
                                            Returns 0 on success, or –1 on error
```

Các hàm `futimes()` và `lutimes()` thực hiện tác vụ tương tự `utimes()` nhưng khác ở cách xác định file:
- `futimes()`: chỉ định file qua file descriptor đã mở
- `lutimes()`: chỉ định qua pathname; nếu là symbolic link, timestamp của chính link được thay đổi (không dereference)

### 15.2.2 Thay Đổi File Timestamp bằng utimensat() và futimens()

System call `utimensat()` (từ kernel 2.6.22) và hàm thư viện `futimens()` (từ glibc 2.6) cung cấp chức năng mở rộng để đặt timestamp:

- Có thể đặt timestamp với độ chính xác nanosecond.
- Có thể đặt timestamp độc lập (từng cái một) — tránh race condition khi cần thay đổi chỉ một timestamp.
- Có thể độc lập đặt một trong hai timestamp về thời gian hiện tại.

```c
#define _XOPEN_SOURCE 700
#include <sys/stat.h>
int utimensat(int dirfd, const char *pathname,
              const struct timespec times[2], int flags);
                                          Returns 0 on success, or –1 on error
```

Nếu `times` là `NULL`, cả hai timestamp được cập nhật thành thời gian hiện tại. Nếu không `NULL`, `times[0]` là new last access timestamp và `times[1]` là new last modification timestamp. Mỗi phần tử là structure `timespec`:

```c
struct timespec {
    time_t tv_sec;  /* Seconds */
    long   tv_nsec; /* Nanoseconds */
};
```

Giá trị đặc biệt `UTIME_NOW` trong `tv_nsec` đặt timestamp về thời gian hiện tại. Giá trị `UTIME_OMIT` để timestamp không thay đổi.

Ví dụ — đặt last access time về hiện tại, không thay đổi last modification time:

```c
struct timespec times[2];
times[0].tv_sec  = 0;
times[0].tv_nsec = UTIME_NOW;
times[1].tv_sec  = 0;
times[1].tv_nsec = UTIME_OMIT;
if (utimensat(AT_FDCWD, "myfile", times, 0) == -1)
    errExit("utimensat");
```

Hàm `futimens()` cập nhật timestamp của file được tham chiếu bởi file descriptor:

```c
#define _GNU_SOURCE
#include <sys/stat.h>
int futimens(int fd, const struct timespec times[2]);
                                             Returns 0 on success, or –1 on error
```

---

## 15.3 Quyền Sở Hữu File

Mỗi file có user ID (UID) và group ID (GID) liên quan. Chúng xác định user và group nào sở hữu file.

### 15.3.1 Quyền Sở Hữu của File Mới

Khi file mới được tạo, user ID của nó lấy từ effective user ID của process. Group ID của file mới có thể lấy từ effective group ID của process (hành vi mặc định System V) hoặc từ group ID của parent directory (hành vi BSD).

Khi file system ext2 được mount:
- Nếu tùy chọn `-o grpid` (hay `-o bsdgroups`) được chỉ định: file mới luôn kế thừa group ID từ parent directory.
- Nếu tùy chọn `-o nogrpid` (hay `-o sysvgroups`, mặc định): file mới lấy group ID từ effective group ID của process. Tuy nhiên, nếu bit set-group-ID được bật cho directory (qua `chmod g+s`), thì group ID của file được kế thừa từ parent directory.

**Bảng 15-3:** Quy tắc xác định group ownership của file mới tạo

| Tùy chọn mount của file system | Bit set-group-ID trên parent dir? | Group ownership lấy từ |
|-------------------------------|----------------------------------|------------------------|
| `-o grpid`, `-o bsdgroups` | (bị bỏ qua) | Group ID của parent directory |
| `-o nogrpid`, `-o sysvgroups` (mặc định) | Không | Effective group ID của process |
| `-o nogrpid`, `-o sysvgroups` (mặc định) | Có | Group ID của parent directory |

### 15.3.2 Thay Đổi Quyền Sở Hữu File: chown(), fchown(), và lchown()

Các system call `chown()`, `lchown()`, và `fchown()` thay đổi owner (user ID) và group (group ID) của file.

```c
#include <unistd.h>
int chown(const char *pathname, uid_t owner, gid_t group);
#define _XOPEN_SOURCE 500
#include <unistd.h>
int lchown(const char *pathname, uid_t owner, gid_t group);
int fchown(int fd, uid_t owner, gid_t group);
                                          All return 0 on success, or –1 on error
```

Sự khác biệt giữa ba system call này tương tự bộ hàm `stat()`:
- `chown()`: thay đổi ownership của file được đặt tên trong `pathname`;
- `lchown()`: tương tự, ngoại trừ nếu `pathname` là symbolic link, ownership của chính link được thay đổi; và
- `fchown()`: thay đổi ownership của file được tham chiếu bởi file descriptor `fd`.

Đối số `owner` chỉ định user ID mới, `group` chỉ định group ID mới. Để chỉ thay đổi một trong hai ID, có thể chỉ định `-1` cho đối số kia.

Chỉ process có đặc quyền (`CAP_CHOWN`) mới có thể dùng `chown()` để thay đổi user ID của file. Process không có đặc quyền có thể dùng `chown()` để thay đổi group ID của file mình sở hữu sang bất kỳ group nào mà họ là thành viên.

Nếu owner hoặc group của file được thay đổi, các bit set-user-ID và set-group-ID đều bị tắt — biện pháp bảo mật ngăn user thường tạo chương trình set-user-ID/set-group-ID sở hữu bởi user có đặc quyền.

**Listing 15-2:** Thay đổi owner và group của file

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––– files/t_chown.c
#include <pwd.h>
#include <grp.h>
#include "ugid_functions.h"
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    uid_t uid;
    gid_t gid;
    int j;
    Boolean errFnd;
    if (argc < 3 || strcmp(argv[1], "--help") == 0)
        usageErr("%s owner group [file...]\n"
                 "  owner or group can be '-', meaning leave unchanged\n",
                 argv[0]);
    if (strcmp(argv[1], "-") == 0) {
        uid = -1;
    } else {
        uid = userIdFromName(argv[1]);
        if (uid == -1)
            fatal("No such user (%s)", argv[1]);
    }
    if (strcmp(argv[2], "-") == 0) {
        gid = -1;
    } else {
        gid = groupIdFromName(argv[2]);
        if (gid == -1)
            fatal("No group user (%s)", argv[1]);
    }
    errFnd = FALSE;
    for (j = 3; j < argc; j++) {
        if (chown(argv[j], uid, gid) == -1) {
            errMsg("chown: %s", argv[j]);
            errFnd = TRUE;
        }
    }
    exit(errFnd ? EXIT_FAILURE : EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––– files/t_chown.c
```

---

## 15.4 Quyền Truy Cập File

Trong mục này, chúng ta mô tả scheme quyền truy cập áp dụng cho file và directory. Mặc dù chủ yếu nói về quyền cho file thông thường và directory, các quy tắc mô tả áp dụng cho tất cả loại file, kể cả device, FIFO, và UNIX domain socket.

### 15.4.1 Quyền Trên File Thông Thường

12 bit dưới của trường `st_mode` định nghĩa quyền truy cập cho file. 3 bit đầu trong số này là các bit đặc biệt: set-user-ID, set-group-ID, và sticky (ký hiệu lần lượt là U, G, T). 9 bit còn lại tạo thành mask định nghĩa quyền được cấp cho các loại người dùng:

- **Owner** (còn gọi là user): Quyền được cấp cho owner của file.
- **Group**: Quyền được cấp cho các user là thành viên của group của file.
- **Other**: Quyền được cấp cho tất cả người khác.

Ba quyền có thể được cấp cho mỗi loại người dùng:
- **Read**: Có thể đọc nội dung file.
- **Write**: Có thể sửa đổi nội dung file.
- **Execute**: File có thể được thực thi.

```
$ ls -l myscript.sh
-rwxr-x--- 1 mtk users 1667 Jan 15 09:22 myscript.sh
```

Chuỗi quyền `rwxr-x---`: nhóm đầu `rwx` cho owner (read, write, execute); nhóm thứ hai `r-x` cho group (read, execute, không write); nhóm cuối `---` cho other (không có quyền nào).

**Bảng 15-4:** Hằng số cho các bit quyền truy cập file

| Hằng số | Giá trị octal | Bit quyền |
|---------|---------------|-----------|
| `S_ISUID` | 04000 | Set-user-ID |
| `S_ISGID` | 02000 | Set-group-ID |
| `S_ISVTX` | 01000 | Sticky |
| `S_IRUSR` | 0400 | User-read |
| `S_IWUSR` | 0200 | User-write |
| `S_IXUSR` | 0100 | User-execute |
| `S_IRGRP` | 040 | Group-read |
| `S_IWGRP` | 020 | Group-write |
| `S_IXGRP` | 010 | Group-execute |
| `S_IROTH` | 04 | Other-read |
| `S_IWOTH` | 02 | Other-write |
| `S_IXOTH` | 01 | Other-execute |

**Listing 15-4:** Chuyển đổi permission mask sang chuỗi

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– files/file_perms.c
#include <sys/stat.h>
#include <stdio.h>
#include "file_perms.h"
#define STR_SIZE sizeof("rwxrwxrwx")
char *
filePermStr(mode_t perm, int flags)
{
    static char str[STR_SIZE];
    snprintf(str, STR_SIZE, "%c%c%c%c%c%c%c%c%c",
        (perm & S_IRUSR) ? 'r' : '-',
        (perm & S_IWUSR) ? 'w' : '-',
        (perm & S_IXUSR) ?
            (((perm & S_ISUID) && (flags & FP_SPECIAL)) ? 's' : 'x') :
            (((perm & S_ISUID) && (flags & FP_SPECIAL)) ? 'S' : '-'),
        (perm & S_IRGRP) ? 'r' : '-',
        (perm & S_IWGRP) ? 'w' : '-',
        (perm & S_IXGRP) ?
            (((perm & S_ISGID) && (flags & FP_SPECIAL)) ? 's' : 'x') :
            (((perm & S_ISGID) && (flags & FP_SPECIAL)) ? 'S' : '-'),
        (perm & S_IROTH) ? 'r' : '-',
        (perm & S_IWOTH) ? 'w' : '-',
        (perm & S_IXOTH) ?
            (((perm & S_ISVTX) && (flags & FP_SPECIAL)) ? 't' : 'x') :
            (((perm & S_ISVTX) && (flags & FP_SPECIAL)) ? 'T' : '-'));
    return str;
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– files/file_perms.c
```

### 15.4.2 Quyền Trên Directory

Directory có cùng scheme quyền như file, nhưng ba quyền được diễn giải khác nhau:

- **Read**: Nội dung (danh sách tên file) của directory có thể được liệt kê (ví dụ: bằng `ls`).
- **Write**: Có thể tạo và xóa file trong directory. Lưu ý không cần có quyền nào trên chính file để có thể xóa nó.
- **Execute**: Các file trong directory có thể được truy cập. Còn gọi là search permission.

Khi truy cập file, cần quyền execute trên tất cả directory trong pathname. Ví dụ, để đọc file `/home/mtk/x` cần quyền execute trên `/`, `/home`, và `/home/mtk`.

Read permission trên directory chỉ cho phép xem danh sách tên file. Phải có quyền execute để truy cập nội dung hoặc thông tin i-node của file trong directory.

### 15.4.3 Thuật Toán Kiểm Tra Quyền

Kernel kiểm tra quyền file bất cứ khi nào ta chỉ định pathname trong system call truy cập file hay directory. Kiểm tra quyền được thực hiện bằng effective user ID, effective group ID, và supplementary group ID của process.

Các quy tắc kernel áp dụng:

1. Nếu process có đặc quyền, tất cả truy cập đều được cấp.
2. Nếu effective user ID của process trùng với user ID (owner) của file, quyền truy cập được cấp theo quyền owner trên file.
3. Nếu effective group ID của process hoặc bất kỳ supplementary group ID nào khớp với group ID của file, quyền truy cập được cấp theo quyền group trên file.
4. Nếu không, quyền truy cập được cấp theo quyền other trên file.

Việc kiểm tra theo thứ tự owner, group, other dừng lại ngay khi quy tắc áp dụng được tìm thấy. Điều này có hệ quả bất ngờ: nếu quyền group vượt quá quyền owner, thì owner thực sự có ít quyền hơn các thành viên group của file.

### 15.4.4 Kiểm Tra Khả Năng Truy Cập File: access()

System call `access()` kiểm tra khả năng truy cập file được chỉ định trong `pathname` dựa trên real user và group ID của process (và supplementary group ID).

```c
#include <unistd.h>
int access(const char *pathname, int mode);
                           Returns 0 if all permissions are granted, otherwise –1
```

**Bảng 15-5:** Hằng số `mode` cho `access()`

| Hằng số | Mô tả |
|---------|-------|
| `F_OK` | File có tồn tại không? |
| `R_OK` | File có thể đọc không? |
| `W_OK` | File có thể ghi không? |
| `X_OK` | File có thể thực thi không? |

Khoảng thời gian giữa lời gọi `access()` và thao tác tiếp theo trên file có nghĩa là không có đảm bảo rằng thông tin trả về bởi `access()` vẫn đúng vào lúc thao tác sau này. Điều này có thể dẫn đến lỗ hổng bảo mật. Thực hành được khuyến nghị là tránh dùng `access()` hoàn toàn — thay vào đó tạm thời thay đổi effective user ID của process, thử thao tác mong muốn, rồi kiểm tra giá trị trả về và `errno`.

### 15.4.5 Bit Set-User-ID, Set-Group-ID, và Sticky

Ngoài 9 bit dùng cho quyền owner, group, và other, permission mask chứa 3 bit bổ sung: set-user-ID (04000), set-group-ID (02000), và sticky (01000). Chúng ta đã thảo luận về set-user-ID và set-group-ID trong Mục 9.3.

Bit set-group-ID cũng phục vụ hai mục đích khác: kiểm soát group ownership của file mới trong directory (Mục 15.3.1) và bật mandatory locking trên file (Mục 55.4).

**Sticky bit**: Trên các triển khai UNIX hiện đại (gồm Linux), sticky bit phục vụ mục đích khác nhau. Với **directory**, sticky bit hoạt động như restricted deletion flag. Khi đặt bit này trên directory, process không có đặc quyền chỉ có thể unlink và rename file trong directory nếu có quyền write trên directory VÀ sở hữu file hoặc directory. Điều này cho phép tạo directory được chia sẻ bởi nhiều người dùng, mỗi người có thể tạo và xóa file của mình nhưng không thể xóa file của người khác. Sticky bit thường được đặt trên thư mục `/tmp`.

```
$ chmod +t tfile
$ ls -l tfile
-rw-r--r-T 1 mtk users 0 Jun 23 14:44 tfile    ← T = sticky, no other-execute
$ chmod o+x tfile
$ ls -l tfile
-rw-r--r-t 1 mtk users 0 Jun 23 14:44 tfile    ← t = sticky, with other-execute
```

### 15.4.6 Process File Mode Creation Mask: umask()

Umask là một thuộc tính process chỉ định các bit quyền nào phải luôn bị tắt khi file hoặc directory mới được tạo bởi process.

Thường thì process chỉ dùng umask kế thừa từ parent shell, với hệ quả mà người dùng có thể kiểm soát umask của chương trình thực thi từ shell bằng lệnh shell built-in `umask`.

File khởi tạo của hầu hết shell đặt umask mặc định là octal `022` (`----w--w-`). Điều này có nghĩa quyền write phải luôn bị tắt cho group và other. Vì vậy, nếu `mode` trong lời gọi `open()` là `0666`, file mới được tạo với quyền `rw-r--r--`.

System call `umask()` thay đổi umask của process về giá trị được chỉ định trong `mask`:

```c
#include <sys/stat.h>
mode_t umask(mode_t mask);
                         Always successfully returns the previous process umask
```

Lời gọi `umask()` luôn thành công và trả về umask cũ.

**Listing 15-5:** Sử dụng `umask()`

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––– files/t_umask.c
#include <sys/stat.h>
#include <fcntl.h>
#include "file_perms.h"
#include "tlpi_hdr.h"
#define MYFILE "myfile"
#define MYDIR  "mydir"
#define FILE_PERMS    (S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP)
#define DIR_PERMS     (S_IRWXU | S_IRWXG | S_IRWXO)
#define UMASK_SETTING (S_IWGRP | S_IXGRP | S_IWOTH | S_IXOTH)
int
main(int argc, char *argv[])
{
    int fd;
    struct stat sb;
    mode_t u;
    umask(UMASK_SETTING);
    fd = open(MYFILE, O_RDWR | O_CREAT | O_EXCL, FILE_PERMS);
    if (fd == -1)  errExit("open-%s", MYFILE);
    if (mkdir(MYDIR, DIR_PERMS) == -1) errExit("mkdir-%s", MYDIR);
    u = umask(0);   /* Lấy (và xóa) giá trị umask */
    if (stat(MYFILE, &sb) == -1) errExit("stat-%s", MYFILE);
    printf("Requested file perms: %s\n",   filePermStr(FILE_PERMS, 0));
    printf("Process umask:        %s\n",   filePermStr(u, 0));
    printf("Actual file perms:    %s\n\n", filePermStr(sb.st_mode, 0));
    if (stat(MYDIR, &sb) == -1) errExit("stat-%s", MYDIR);
    printf("Requested dir. perms: %s\n", filePermStr(DIR_PERMS, 0));
    printf("Process umask:        %s\n", filePermStr(u, 0));
    printf("Actual dir. perms:    %s\n", filePermStr(sb.st_mode, 0));
    if (unlink(MYFILE) == -1) errMsg("unlink-%s", MYFILE);
    if (rmdir(MYDIR)  == -1) errMsg("rmdir-%s", MYDIR);
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––– files/t_umask.c
```

### 15.4.7 Thay Đổi Quyền File: chmod() và fchmod()

Các system call `chmod()` và `fchmod()` thay đổi quyền của file.

```c
#include <sys/stat.h>
int chmod(const char *pathname, mode_t mode);
#define _XOPEN_SOURCE 500
#include <sys/stat.h>
int fchmod(int fd, mode_t mode);
                                        Both return 0 on success, or –1 on error
```

`chmod()` thay đổi quyền của file được đặt tên. Nếu là symbolic link, thay đổi quyền của file được link trỏ đến. `fchmod()` thay đổi quyền của file được tham chiếu bởi file descriptor `fd`.

Đối số `mode` chỉ định quyền mới (dạng số octal hoặc OR các hằng số từ Bảng 15-4). Để thay đổi quyền, process phải có đặc quyền (`CAP_FOWNER`) hoặc effective user ID phải khớp với owner của file.

Ví dụ:
```c
/* Chỉ cấp quyền read cho tất cả user */
if (chmod("myfile", S_IRUSR | S_IRGRP | S_IROTH) == -1)
    errExit("chmod");
/* Tương đương: chmod("myfile", 0444); */

/* Bật owner-write, tắt other-read */
struct stat sb;
mode_t mode;
if (stat("myfile", &sb) == -1)
    errExit("stat");
mode = (sb.st_mode | S_IWUSR) & ~S_IROTH;
if (chmod("myfile", mode) == -1)
    errExit("chmod");
/* Tương đương: chmod u+w,o-r myfile */
```

---

## 15.5 I-node Flag (ext2 Extended File Attribute)

Một số Linux file system cho phép đặt các i-node flag trên file và directory. Tính năng này là phần mở rộng không chuẩn của Linux.

File system Linux đầu tiên hỗ trợ i-node flag là ext2. Sau đó, hỗ trợ i-node flag được thêm vào các file system khác, gồm Btrfs, ext3, ext4, Reiserfs, XFS, và JFS.

Từ shell, i-node flag có thể đặt và xem bằng lệnh `chattr` và `lsattr`:

```
$ lsattr myfile
-------- myfile
$ chattr +ai myfile    Bật Append Only và Immutable flag
$ lsattr myfile
----ia-- myfile
```

Trong chương trình, i-node flag có thể lấy và sửa đổi bằng system call `ioctl()`.

**Bảng 15-6:** I-node flag

| Hằng số | Tùy chọn chattr | Mục đích |
|---------|----------------|---------|
| `FS_APPEND_FL` | `a` | Chỉ append (cần đặc quyền) |
| `FS_COMPR_FL` | `c` | Bật nén file (chưa triển khai) |
| `FS_DIRSYNC_FL` | `D` | Cập nhật directory đồng bộ (từ Linux 2.6) |
| `FS_IMMUTABLE_FL` | `i` | Immutable (cần đặc quyền) |
| `FS_JOURNAL_DATA_FL` | `j` | Bật journaling dữ liệu (cần đặc quyền) |
| `FS_NOATIME_FL` | `A` | Không cập nhật last access time |
| `FS_NODUMP_FL` | `d` | Không dump |
| `FS_NOTAIL_FL` | `t` | Không tail packing |
| `FS_SECRM_FL` | `s` | Xóa an toàn (chưa triển khai) |
| `FS_SYNC_FL` | `S` | Cập nhật file (và directory) đồng bộ |
| `FS_TOPDIR_FL` | `T` | Coi là top-level directory cho Orlov (từ Linux 2.6) |
| `FS_UNRM_FL` | `u` | File có thể khôi phục (chưa triển khai) |

Mô tả các flag chính:

- **`FS_APPEND_FL`**: File chỉ có thể mở để ghi nếu flag `O_APPEND` được chỉ định. Hữu ích cho log file.
- **`FS_IMMUTABLE_FL`**: Làm file bất biến — dữ liệu không thể cập nhật, thay đổi metadata bị ngăn chặn. Ngay cả process có đặc quyền cũng không thể thay đổi.
- **`FS_JOURNAL_DATA_FL`**: Bật journaling dữ liệu (chỉ cho ext3 và ext4).
- **`FS_NOATIME_FL`**: Không cập nhật last access time khi file được truy cập.
- **`FS_SYNC_FL`**: Làm cho ghi file đồng bộ (như thể `O_SYNC` được chỉ định khi mở).
- **`FS_DIRSYNC_FL`**: Làm cho cập nhật directory đồng bộ.

Để lấy và sửa đổi i-node flag trong chương trình, dùng các thao tác `FS_IOC_GETFLAGS` và `FS_IOC_SETFLAGS` của `ioctl()`:

```c
int attr;
if (ioctl(fd, FS_IOC_GETFLAGS, &attr) == -1)  /* Lấy flag hiện tại */
    errExit("ioctl");
attr |= FS_NOATIME_FL;
if (ioctl(fd, FS_IOC_SETFLAGS, &attr) == -1)  /* Cập nhật flag */
    errExit("ioctl");
```

---

## 15.6 Tóm Tắt

System call `stat()` lấy thông tin về file (metadata), chủ yếu từ i-node của file. Thông tin này gồm quyền sở hữu file, quyền truy cập file, và file timestamp.

Chương trình có thể cập nhật last access time và last modification time của file bằng `utime()`, `utimes()`, và các giao diện tương tự.

Mỗi file có user ID (owner) và group ID đi kèm, cũng như tập bit quyền. Với mục đích quyền, người dùng file được chia thành ba loại: owner, group, và other. Ba quyền có thể được cấp cho mỗi loại: read, write, và execute. Cùng scheme được dùng với directory, mặc dù các bit quyền có nghĩa hơi khác một chút. Các system call `chown()` và `chmod()` thay đổi quyền sở hữu và quyền truy cập file. System call `umask()` đặt mask các bit quyền luôn bị tắt khi process gọi tạo file.

Ba bit quyền bổ sung được dùng cho file và directory: set-user-ID, set-group-ID, và sticky bit. Khi áp dụng cho directory, sticky bit hoạt động như restricted deletion flag.

I-node flag kiểm soát các hành vi khác nhau của file và directory.

---

## 15.7 Bài Tập

- **15-1.** Kiểm tra các nhận định về quyền trong Mục 15.4: (a) Xóa tất cả quyền owner từ file từ chối quyền truy cập owner, ngay cả khi group và other có quyền; (b) Trên directory chỉ có quyền read nhưng không có execute, tên file có thể liệt kê nhưng bản thân file không thể truy cập; (c) Quyền cần thiết để tạo file mới, mở để đọc, ghi, và xóa file.
- **15-2.** Bạn có mong đợi bất kỳ timestamp nào trong ba timestamp của file bị thay đổi bởi system call `stat()` không? Giải thích tại sao.
- **15-3.** Trên hệ thống chạy Linux 2.6, sửa đổi chương trình trong Listing 15-1 để hiển thị file timestamp với độ chính xác nanosecond.
- **15-4.** System call `access()` kiểm tra quyền bằng real user và group ID của process. Viết hàm tương ứng kiểm tra theo effective user và group ID của process.
- **15-5.** Như đã nói trong Mục 15.4.6, `umask()` luôn đặt umask của process và đồng thời trả về bản sao umask cũ. Làm thế nào để lấy bản sao umask hiện tại trong khi để nguyên nó không thay đổi?
- **15-6.** Viết chương trình dùng `stat()` và `chmod()` để thực hiện tương đương của `chmod a+rX`.
- **15-7.** Viết phiên bản đơn giản của lệnh `chattr(1)`.
