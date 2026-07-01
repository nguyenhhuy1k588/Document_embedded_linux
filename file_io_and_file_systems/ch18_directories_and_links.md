## Chương 18
# Directory và Link

Trong chương này, chúng ta kết thúc thảo luận về các chủ đề liên quan đến file bằng cách xem xét directory và link. Sau tổng quan về cách triển khai của chúng, chúng ta mô tả các system call dùng để tạo và xóa directory và link, sau đó xem các hàm thư viện cho phép chương trình quét nội dung của một directory và duyệt qua cây directory. Kết thúc chương là thảo luận về các hàm thư viện dùng để phân giải pathname và phân tách chúng thành các thành phần directory và filename.

---

## 18.1 Directory và Hard Link

Directory được lưu trong file system tương tự như file thông thường. Hai điều phân biệt directory với file thông thường:

- Directory được đánh dấu với loại file khác trong i-node entry của nó (Mục 14.4).
- Directory là file với tổ chức đặc biệt — về cơ bản là bảng gồm filename và i-node number.

Trên hầu hết file system gốc của Linux, filename có thể dài đến 255 ký tự.

I-node không chứa filename của file; chỉ có việc ánh xạ trong danh sách directory mới định nghĩa tên của file. Điều này có hệ quả hữu ích: ta có thể tạo nhiều tên — trong cùng hoặc trong các directory khác nhau — mỗi tên đều tham chiếu đến cùng một i-node. Các tên nhiều này được gọi là link, hay đôi khi là **hard link** để phân biệt với symbolic link.

Từ shell, ta có thể tạo hard link mới đến file hiện có bằng lệnh `ln`:

```
$ echo -n 'It is good to collect things,' > abc
$ ls -li abc
 122232 -rw-r--r-- 1 mtk users 29 Jun 15 17:07 abc
$ ln abc xyz
$ echo ' but it is better to go on walks.' >> xyz
$ cat abc
It is good to collect things, but it is better to go on walks.
$ ls -li abc xyz
 122232 -rw-r--r-- 2 mtk users 63 Jun 15 17:07 abc
 122232 -rw-r--r-- 2 mtk users 63 Jun 15 17:07 xyz
```

I-node number được hiển thị (là cột đầu tiên) bởi `ls -li` xác nhận rằng tên `abc` và `xyz` đều tham chiếu đến cùng i-node. Link count của i-node đã tăng lên 2. Nếu một trong các filename được xóa, filename kia và bản thân file tiếp tục tồn tại:

```
$ rm abc
$ ls -li xyz
 122232 -rw-r--r-- 1 mtk users 63 Jun 15 17:07 xyz
```

I-node entry và data block của file chỉ bị xóa (deallocate) khi link count của i-node xuống về 0. Lệnh `rm` xóa filename khỏi danh sách directory, giảm link count đi 1, và nếu link count xuống 0 thì deallocate i-node và data block.

Hard link có hai hạn chế, cả hai đều có thể được khắc phục bằng symbolic link:

- Vì directory entry (hard link) tham chiếu file bằng i-node number, và i-node number chỉ là duy nhất trong một file system, hard link phải nằm trên cùng file system với file mà nó tham chiếu.
- Không thể tạo hard link đến directory. Điều này ngăn việc tạo circular link.

---

## 18.2 Symbolic (Soft) Link

Symbolic link, đôi khi còn gọi là soft link, là loại file đặc biệt có dữ liệu là tên của một file khác.

Từ shell, symbolic link được tạo bằng lệnh `ln -s`. Lệnh `ls -F` hiển thị ký tự `@` ở cuối symbolic link.

Pathname mà symbolic link tham chiếu có thể là tuyệt đối hoặc tương đối. Symbolic link tương đối được diễn giải tương đối với vị trí của bản thân link.

Symbolic link không được tính vào link count của file mà nó tham chiếu. Do đó, nếu filename mà symbolic link tham chiếu bị xóa, bản thân symbolic link vẫn tồn tại, mặc dù nó không thể được dereference (follow) nữa — được gọi là **dangling link**.

Symbolic link khác hard link ở các điểm quan trọng:
- Vì tham chiếu đến filename thay vì i-node number, symbolic link có thể link đến file trên file system khác.
- Có thể tạo symbolic link đến directory.

**Diễn giải symbolic link bởi system call**: Nhiều system call dereference (follow) symbolic link và do đó làm việc trên file mà link tham chiếu. Một số system call không dereference symbolic link mà thay vào đó thao tác trực tiếp trên chính file link. Bảng 18-1 tóm tắt hành vi này.

**Quyền và ownership cho symbolic link**: Ownership và quyền của symbolic link bị bỏ qua đối với hầu hết các thao tác (symbolic link luôn được tạo với tất cả quyền được bật). Thay vào đó, ownership và quyền của file mà link tham chiếu được dùng để xác định thao tác có được phép không.

**Bảng 18-1:** Diễn giải symbolic link bởi các hàm khác nhau

| Hàm | Follow link? | Ghi chú |
|-----|-------------|---------|
| `access()` | Có | |
| `chdir()` | Có | |
| `chmod()` | Có | |
| `chown()` | Có | |
| `lchown()` | Không | |
| `link()` | Không | Xem Mục 18.3 |
| `lstat()` | Không | |
| `lutimes()` | Không | |
| `open()` | Có | Trừ khi `O_NOFOLLOW` hoặc `O_EXCL|O_CREAT` được chỉ định |
| `readlink()` | Không | |
| `rename()` | Không | Link không được follow ở cả hai đối số |
| `rmdir()` | Không | Thất bại với `ENOTDIR` nếu đối số là symbolic link |
| `stat()` | Có | |
| `truncate()` | Có | |
| `unlink()` | Không | |

---

## 18.3 Tạo và Xóa Hard Link: link() và unlink()

System call `link()` và `unlink()` tạo và xóa hard link.

```c
#include <unistd.h>
int link(const char *oldpath, const char *newpath);
                                            Returns 0 on success, or –1 on error
```

Với pathname của file hiện có trong `oldpath`, `link()` tạo link mới bằng pathname được chỉ định trong `newpath`. Nếu `newpath` đã tồn tại, không bị ghi đè; thay vào đó lỗi `EEXIST`.

Trên Linux, `link()` không dereference symbolic link — nếu `oldpath` là symbolic link, `newpath` được tạo như hard link mới đến cùng file symbolic link.

```c
#include <unistd.h>
int unlink(const char *pathname);
                                            Returns 0 on success, or –1 on error
```

`unlink()` xóa link (xóa filename) và nếu đó là link cuối cùng đến file thì cũng xóa bản thân file. Nếu link được chỉ định trong `pathname` không tồn tại, `unlink()` thất bại với `ENOENT`.

Không thể dùng `unlink()` để xóa directory; tác vụ đó yêu cầu `rmdir()` hoặc `remove()`.

`unlink()` không dereference symbolic link — nếu `pathname` là symbolic link, chính link được xóa thay vì tên mà nó trỏ đến.

### File đang mở chỉ bị xóa khi tất cả file descriptor được đóng

Kernel cũng đếm số open file description cho file. Nếu link cuối cùng đến file bị xóa và bất kỳ process nào giữ file descriptor đang mở tham chiếu đến file, file sẽ không thực sự bị xóa cho đến khi tất cả descriptor được đóng. Đây là tính năng hữu ích: cho phép unlink file mà không cần lo lắng về việc process nào đó có đang mở nó không. Ngoài ra, ta có thể tạo và mở file tạm, unlink ngay lập tức, và tiếp tục dùng nó trong chương trình — file bị xóa khi ta đóng file descriptor.

---

## 18.4 Đổi Tên File: rename()

System call `rename()` có thể dùng để đổi tên file và di chuyển nó sang directory khác trên cùng file system.

```c
#include <stdio.h>
int rename(const char *oldpath, const char *newpath);
                                            Returns 0 on success, or –1 on error
```

`rename()` chỉ thao tác directory entry; không di chuyển file data. Các quy tắc áp dụng:

- Nếu `newpath` đã tồn tại, nó bị ghi đè.
- Nếu `newpath` và `oldpath` tham chiếu cùng file, không có thay đổi nào được thực hiện (và lời gọi thành công).
- `rename()` không dereference symbolic link ở cả hai đối số.
- Nếu `oldpath` tham chiếu file không phải directory, `newpath` không thể chỉ định pathname của directory.
- Nếu `oldpath` là directory, `newpath` phải không tồn tại hoặc phải là tên của directory rỗng.
- Nếu `oldpath` là directory, `newpath` không thể chứa prefix directory giống `oldpath`.
- Các file được tham chiếu bởi `oldpath` và `newpath` phải trên cùng file system (vì `rename()` chỉ thao tác danh sách directory).

---

## 18.5 Làm Việc với Symbolic Link: symlink() và readlink()

System call `symlink()` tạo symbolic link mới `linkpath` đến pathname được chỉ định trong `filepath`.

```c
#include <unistd.h>
int symlink(const char *filepath, const char *linkpath);
                                             Returns 0 on success, or –1 on error
```

Nếu pathname trong `linkpath` đã tồn tại, lời gọi thất bại với `EEXIST`. File hoặc directory được đặt tên trong `filepath` không cần tồn tại vào lúc gọi.

Khi ta chỉ định symbolic link làm đối số `pathname` cho `open()`, nó mở file mà link tham chiếu. Đôi khi ta muốn lấy nội dung của chính link — tức là pathname mà nó tham chiếu. System call `readlink()` thực hiện tác vụ này:

```c
#include <unistd.h>
ssize_t readlink(const char *pathname, char *buffer, size_t bufsiz);
             Returns number of bytes placed in buffer on success, or –1 on error
```

`readlink()` đặt bản sao chuỗi symbolic link vào mảng ký tự được trỏ bởi `buffer`. Chú ý: không có null byte kết thúc được đặt ở cuối buffer.

---

## 18.6 Tạo và Xóa Directory: mkdir() và rmdir()

System call `mkdir()` tạo directory mới.

```c
#include <sys/stat.h>
int mkdir(const char *pathname, mode_t mode);
                                            Returns 0 on success, or –1 on error
```

Quyền sở hữu của directory mới được đặt theo quy tắc mô tả trong Mục 15.3.1. Đối số `mode` chỉ định quyền và được AND với process umask. Directory mới chứa hai entry: `.` (dot, link đến chính directory) và `..` (dot-dot, link đến parent directory).

`mkdir()` chỉ tạo thành phần cuối của pathname — tức là lời gọi `mkdir("aaa/bbb/ccc", mode)` chỉ thành công nếu `aaa` và `aaa/bbb` đã tồn tại.

System call `rmdir()` xóa directory được chỉ định trong `pathname`.

```c
#include <unistd.h>
int rmdir(const char *pathname);
                                            Returns 0 on success, or –1 on error
```

Để `rmdir()` thành công, directory phải rỗng. Nếu thành phần cuối của `pathname` là symbolic link, không được dereference; thay vào đó lỗi `ENOTDIR`.

---

## 18.7 Xóa File hoặc Directory: remove()

Hàm thư viện `remove()` xóa file hoặc directory rỗng.

```c
#include <stdio.h>
int remove(const char *pathname);
                                             Returns 0 on success, or –1 on error
```

Nếu `pathname` là file, `remove()` gọi `unlink()`; nếu là directory, `remove()` gọi `rmdir()`. Giống `unlink()` và `rmdir()`, `remove()` không dereference symbolic link — xóa chính link.

---

## 18.8 Đọc Directory: opendir() và readdir()

### opendir() và fdopendir()

Hàm `opendir()` mở directory và trả về handle để sử dụng trong các lời gọi sau.

```c
#include <dirent.h>
DIR *opendir(const char *dirpath);
                               Returns directory stream handle, or NULL on error
```

Trả về pointer đến structure kiểu `DIR` (directory stream). Khi trả về từ `opendir()`, directory stream được định vị ở entry đầu tiên trong danh sách directory.

Hàm `fdopendir()` tương tự `opendir()`, nhưng directory được chỉ định qua file descriptor `fd` đang mở.

```c
#include <dirent.h>
DIR *fdopendir(int fd);
                               Returns directory stream handle, or NULL on error
```

### readdir()

Hàm `readdir()` đọc các entry liên tiếp từ directory stream.

```c
#include <dirent.h>
struct dirent *readdir(DIR *dirp);
                     Returns pointer to a statically allocated structure describing
                        next directory entry, or NULL on end-of-directory or error
```

Mỗi lời gọi `readdir()` đọc entry tiếp theo từ directory stream được tham chiếu bởi `dirp` và trả về pointer đến structure `dirent` được cấp phát tĩnh:

```c
struct dirent {
    ino_t d_ino;    /* File i-node number */
    char  d_name[]; /* Null-terminated name of file */
};
```

Structure này bị ghi đè mỗi lần gọi. Tên file được trả về không theo thứ tự sắp xếp.

Khi hết directory hoặc lỗi, `readdir()` trả về `NULL`. Để phân biệt:

```c
errno = 0;
direntp = readdir(dirp);
if (direntp == NULL) {
    if (errno != 0) {
        /* Xử lý lỗi */
    } else {
        /* Đã đến end-of-directory */
    }
}
```

### rewinddir(), closedir(), dirfd()

```c
#include <dirent.h>
void rewinddir(DIR *dirp);      /* Tua lại về đầu */
int  closedir(DIR *dirp);       /* Đóng directory stream */
int  dirfd(DIR *dirp);          /* Lấy file descriptor liên kết */
```

### Chương trình ví dụ

**Listing 18-2:** Liệt kê nội dung directory

```c
––––––––––––––––––––––––––––––––––––––––––––––––––– dirs_links/list_files.c
#include <dirent.h>
#include "tlpi_hdr.h"
static void
listFiles(const char *dirpath)
{
    DIR *dirp;
    struct dirent *dp;
    Boolean isCurrent;
    isCurrent = strcmp(dirpath, ".") == 0;
    dirp = opendir(dirpath);
    if (dirp == NULL) {
        errMsg("opendir failed on '%s'", dirpath);
        return;
    }
    for (;;) {
        errno = 0;
        dp = readdir(dirp);
        if (dp == NULL)
            break;
        if (strcmp(dp->d_name, ".") == 0 || strcmp(dp->d_name, "..") == 0)
            continue;   /* Bỏ qua . và .. */
        if (!isCurrent)
            printf("%s/", dirpath);
        printf("%s\n", dp->d_name);
    }
    if (errno != 0)     errExit("readdir");
    if (closedir(dirp) == -1) errMsg("closedir");
}
int
main(int argc, char *argv[])
{
    if (argc > 1 && strcmp(argv[1], "--help") == 0)
        usageErr("%s [dir...]\n", argv[0]);
    if (argc == 1)
        listFiles(".");
    else
        for (argv++; *argv; argv++)
            listFiles(*argv);
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––– dirs_links/list_files.c
```

### Hàm readdir_r()

`readdir_r()` là biến thể reentrant của `readdir()` — trả về thông tin qua đối số `entry` do caller cấp phát thay vì dùng structure được cấp phát tĩnh.

```c
#include <dirent.h>
int readdir_r(DIR *dirp, struct dirent *entry, struct dirent **result);
                      Returns 0 on success, or a positive error number on error
```

---

## 18.9 Duyệt Cây File: nftw()

Hàm `nftw()` cho phép chương trình duyệt đệ quy toàn bộ directory subtree, thực hiện một thao tác (gọi hàm do lập trình viên định nghĩa) cho mỗi file trong subtree.

```c
#define _XOPEN_SOURCE 500
#include <ftw.h>
int nftw(const char *dirpath,
         int (*func) (const char *pathname, const struct stat *statbuf,
                      int typeflag, struct FTW *ftwbuf),
         int nopenfd, int flags);
                  Returns 0 after successful walk of entire tree, or –1 on error,
                           or the first nonzero value returned by a call to func
```

Theo mặc định, `nftw()` thực hiện preorder traversal không được sắp xếp của cây đã cho, xử lý mỗi directory trước khi xử lý các file và subdirectory trong directory đó.

`nopenfd` chỉ định số file descriptor tối đa mà `nftw()` có thể dùng.

**Flags cho `nftw()`:**
- `FTW_CHDIR`: `chdir()` vào mỗi directory trước khi xử lý nội dung.
- `FTW_DEPTH`: Thực hiện postorder traversal — gọi `func` trên tất cả file trong directory trước khi gọi trên bản thân directory.
- `FTW_MOUNT`: Không vượt sang file system khác.
- `FTW_PHYS`: Không dereference symbolic link.

Với mỗi file, `nftw()` truyền bốn đối số khi gọi `func`:
1. `pathname`: Pathname của file.
2. `statbuf`: Pointer đến structure `stat` chứa thông tin về file.
3. `typeflag`: Thông tin bổ sung về file, một trong: `FTW_D` (directory), `FTW_DNR` (directory không đọc được), `FTW_DP` (directory trong postorder), `FTW_F` (file không phải directory hay symlink), `FTW_NS` (stat() thất bại), `FTW_SL` (symbolic link, chỉ với `FTW_PHYS`), `FTW_SLN` (dangling symlink).
4. `ftwbuf`: Pointer đến structure `FTW`:

```c
struct FTW {
    int base;   /* Offset đến phần basename của pathname */
    int level;  /* Độ sâu của file trong tree traversal */
};
```

Mỗi lần `func` được gọi, nó phải trả về giá trị integer: 0 để tiếp tục, khác 0 để dừng ngay lập tức (và `nftw()` trả về cùng giá trị đó).

**Listing 18-3:** Sử dụng `nftw()` để duyệt cây directory

```c
––––––––––––––––––––––––––––––––––––––––––––––– dirs_links/nftw_dir_tree.c
#define _XOPEN_SOURCE 600
#include <ftw.h>
#include "tlpi_hdr.h"
static int
dirTree(const char *pathname, const struct stat *sbuf, int type,
        struct FTW *ftwb)
{
    switch (sbuf->st_mode & S_IFMT) {
    case S_IFREG:  printf("-"); break;
    case S_IFDIR:  printf("d"); break;
    case S_IFCHR:  printf("c"); break;
    case S_IFBLK:  printf("b"); break;
    case S_IFLNK:  printf("l"); break;
    case S_IFIFO:  printf("p"); break;
    case S_IFSOCK: printf("s"); break;
    default:       printf("?"); break;
    }
    printf(" %s ",
           (type == FTW_D)   ? "D  " : (type == FTW_DNR) ? "DNR" :
           (type == FTW_DP)  ? "DP " : (type == FTW_F)   ? "F  " :
           (type == FTW_SL)  ? "SL " : (type == FTW_SLN) ? "SLN" :
           (type == FTW_NS)  ? "NS " : "   ");
    if (type != FTW_NS)
        printf("%7ld ", (long) sbuf->st_ino);
    else
        printf("        ");
    printf(" %*s", 4 * ftwb->level, "");
    printf("%s\n", &pathname[ftwb->base]);
    return 0;
}
int
main(int argc, char *argv[])
{
    int flags = 0, opt;
    while ((opt = getopt(argc, argv, "dmp")) != -1) {
        switch (opt) {
        case 'd': flags |= FTW_DEPTH; break;
        case 'm': flags |= FTW_MOUNT; break;
        case 'p': flags |= FTW_PHYS;  break;
        default:  usageError(argv[0], NULL);
        }
    }
    if (argc > optind + 1)
        usageError(argv[0], NULL);
    if (nftw((argc > optind) ? argv[optind] : ".", dirTree, 10, flags) == -1) {
        perror("nftw");
        exit(EXIT_FAILURE);
    }
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––– dirs_links/nftw_dir_tree.c
```

---

## 18.10 Current Working Directory của Process

Current working directory của process định nghĩa điểm xuất phát để phân giải relative pathname.

### Lấy current working directory

```c
#include <unistd.h>
char *getcwd(char *cwdbuf, size_t size);
                                      Returns cwdbuf on success, or NULL on error
```

`getcwd()` đặt chuỗi null-terminated chứa absolute pathname của current working directory vào buffer được trỏ bởi `cwdbuf`. Buffer phải được caller cấp phát ít nhất `size` byte. Thông thường kích thước bằng `PATH_MAX`.

### Thay đổi current working directory

System call `chdir()` thay đổi current working directory của process gọi sang pathname được chỉ định.

```c
#include <unistd.h>
int chdir(const char *pathname);
                                             Returns 0 on success, or –1 on error
```

System call `fchdir()` tương tự nhưng directory được chỉ định qua file descriptor đã mở trước đó.

```c
#define _XOPEN_SOURCE 500
#include <unistd.h>
int fchdir(int fd);
                                            Returns 0 on success, or –1 on error
```

Dùng `fchdir()` để thay đổi CWD và sau đó quay lại vị trí ban đầu:

```c
int fd;
fd = open(".", O_RDONLY);  /* Ghi nhớ vị trí hiện tại */
chdir(somepath);            /* Đến nơi khác */
fchdir(fd);                 /* Quay lại directory ban đầu */
close(fd);
```

---

## 18.11 Thao Tác Tương Đối với Directory File Descriptor

Từ kernel 2.6.16, Linux cung cấp một loạt system call mới thực hiện tác vụ tương tự như các system call truyền thống, nhưng có thêm chức năng. Các system call này thêm đối số `dirfd` cho phép diễn giải relative pathname tương đối với directory được chỉ định bởi file descriptor.

**Bảng 18-2:** Các system call mới và tương tự truyền thống

| Giao diện mới | Tương tự truyền thống | Ghi chú |
|--------------|----------------------|---------|
| `faccessat()` | `access()` | Hỗ trợ `AT_EACCESS` và `AT_SYMLINK_NOFOLLOW` |
| `fchmodat()` | `chmod()` | |
| `fchownat()` | `chown()` | Hỗ trợ `AT_SYMLINK_NOFOLLOW` |
| `fstatat()` | `stat()` | Hỗ trợ `AT_SYMLINK_NOFOLLOW` |
| `linkat()` | `link()` | Hỗ trợ `AT_SYMLINK_FOLLOW` |
| `mkdirat()` | `mkdir()` | |
| `openat()` | `open()` | |
| `readlinkat()` | `readlink()` | |
| `renameat()` | `rename()` | |
| `symlinkat()` | `symlink()` | |
| `unlinkat()` | `unlink()` | Hỗ trợ `AT_REMOVEDIR` |
| `utimensat()` | `utimes()` | Hỗ trợ `AT_SYMLINK_NOFOLLOW` |

Ví dụ với `openat()`:

```c
#define _XOPEN_SOURCE 700
#include <fcntl.h>
int openat(int dirfd, const char *pathname, int flags, ... /* mode_t mode */);
                               Returns file descriptor on success, or –1 on error
```

- Nếu `pathname` là relative, nó được diễn giải tương đối với directory được tham chiếu bởi `dirfd`.
- Nếu `dirfd` là `AT_FDCWD`, `pathname` được diễn giải tương đối với CWD của process.
- Nếu `pathname` là tuyệt đối, `dirfd` bị bỏ qua.

Các system call này hữu ích để tránh race condition có thể xảy ra khi dùng `open()` để mở file ở các vị trí khác ngoài CWD, và để các thread khác nhau có "virtual" working directory khác nhau.

---

## 18.12 Thay Đổi Root Directory của Process: chroot()

Mỗi process có một root directory — điểm từ đó absolute pathname được diễn giải. Theo mặc định, đây là root directory thực của file system. Process có đặc quyền (`CAP_SYS_CHROOT`) có thể thay đổi root directory của nó bằng system call `chroot()`.

```c
#define _BSD_SOURCE
#include <unistd.h>
int chroot(const char *pathname);
                                            Returns 0 on success, or –1 on error
```

`chroot()` thay đổi root directory của process sang directory được chỉ định bởi `pathname`. Sau đó, tất cả absolute pathname được diễn giải là bắt đầu từ vị trí đó trong file system — đôi khi gọi là thiết lập **chroot jail**.

Ví dụ điển hình về dùng `chroot()` là trong chương trình FTP: khi user đăng nhập ẩn danh, `ftp` dùng `chroot()` để đặt root directory cho process mới thành directory được dành riêng cho anonymous login.

**Lưu ý bảo mật**: `chroot()` không được coi là cơ chế jail hoàn toàn bảo mật. `chroot()` không thay đổi CWD của process, vì vậy thường phải theo sau bằng `chdir("/")`. Nếu process giữ file descriptor đang mở đến directory ngoài jail, tổ hợp `fchdir()` + `chroot()` có thể dùng để thoát khỏi jail.

---

## 18.13 Phân Giải Pathname: realpath()

Hàm thư viện `realpath()` dereferences tất cả symbolic link trong `pathname` và phân giải tất cả tham chiếu đến `/.` và `/..` để tạo ra chuỗi null-terminated chứa absolute pathname tương ứng.

```c
#include <stdlib.h>
char *realpath(const char *pathname, char *resolved_path);
             Returns pointer to resolved pathname on success, or NULL on error
```

Kết quả được đặt trong buffer được trỏ bởi `resolved_path`, phải là mảng ký tự ít nhất `PATH_MAX` byte.

```
$ pwd
/home/mtk
$ touch x
$ ln -s x y
$ ./view_symlink y
readlink: y --> x
realpath:  y --> /home/mtk/x
```

---

## 18.14 Phân Tách Chuỗi Pathname: dirname() và basename()

Các hàm `dirname()` và `basename()` phân tách chuỗi pathname thành phần directory và filename.

```c
#include <libgen.h>
char *dirname(char *pathname);
char *basename(char *pathname);
                         Both return a pointer to a null-terminated string
```

Ví dụ với pathname `/home/britta/prog.c`: `dirname()` trả về `/home/britta` và `basename()` trả về `prog.c`.

**Bảng 18-3:** Ví dụ chuỗi trả về bởi `dirname()` và `basename()`

| Chuỗi pathname | `dirname()` | `basename()` |
|----------------|-------------|--------------|
| `/` | `/` | `/` |
| `/usr/bin/zip` | `/usr/bin` | `zip` |
| `/etc/passwd////` | `/etc` | `passwd` |
| `etc/passwd` | `etc` | `passwd` |
| `passwd` | `.` | `passwd` |
| `NULL` | `.` | `.` |

Cả `dirname()` và `basename()` có thể sửa đổi chuỗi được trỏ bởi `pathname`. Do đó, nếu muốn bảo toàn chuỗi pathname, phải truyền bản sao cho chúng.

---

## 18.15 Tóm Tắt

I-node không chứa tên file. Thay vào đó, file được gán tên qua các entry trong directory — các entry này được gọi là hard link. Một file có thể có nhiều link, tất cả đều có địa vị ngang nhau. Link được tạo và xóa bằng `link()` và `unlink()`. File có thể đổi tên bằng `rename()`.

Symbolic link được tạo bằng `symlink()`. Symbolic link tương tự hard link ở một số khía cạnh, với điểm khác biệt là symbolic link có thể vượt qua ranh giới file system và có thể tham chiếu đến directory. Symbolic link không được tính trong link count của i-node đích. Một số system call tự động dereference symbolic link; một số thì không.

Directory được tạo bằng `mkdir()` và xóa bằng `rmdir()`. Để quét nội dung directory, ta dùng `opendir()`, `readdir()`, và các hàm liên quan. Hàm `nftw()` cho phép chương trình duyệt cây directory, gọi hàm do lập trình viên định nghĩa để thao tác trên mỗi file.

Mỗi process có root directory và current working directory. `chroot()` và `chdir()` được dùng để thay đổi các thuộc tính này. `getcwd()` trả về CWD của process.

Linux cung cấp các system call (như `openat()`) cho phép diễn giải relative pathname tương đối với directory được chỉ định bởi file descriptor.

`realpath()` phân giải pathname — dereference tất cả symbolic link. `dirname()` và `basename()` phân tách pathname thành các thành phần directory và filename.

---

## 18.16 Bài Tập

- **18-1.** Trong Mục 4.3.2, ta thấy không thể mở file để ghi nếu nó đang được thực thi. Tuy nhiên, ta có thể biên dịch và ghi đè lên file thực thi đang chạy. Tại sao điều này có thể thực hiện được?
- **18-2.** Tại sao lời gọi `chmod()` trong đoạn code sau thất bại?
- **18-3.** Triển khai `realpath()`.
- **18-4.** Sửa đổi chương trình trong Listing 18-2 để dùng `readdir_r()` thay vì `readdir()`.
- **18-5.** Triển khai hàm thực hiện tương đương của `getcwd()`.
- **18-6.** Sửa đổi chương trình trong Listing 18-3 để dùng flag `FTW_DEPTH`.
- **18-7.** Viết chương trình dùng `nftw()` để duyệt cây directory và in ra số lượng và phần trăm của các loại file khác nhau.
- **18-8.** Triển khai `nftw()`.
- **18-9.** Hai kỹ thuật (dùng `fchdir()` và `chdir()`) để quay lại CWD trước đó — cái nào hiệu quả hơn?
