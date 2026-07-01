## Chương 19
# Giám Sát Sự Kiện File (Monitoring File Events)

Một số ứng dụng cần theo dõi file hoặc directory để xác định khi nào sự kiện xảy ra trên các đối tượng được giám sát. Ví dụ, file manager đồ họa cần biết khi nào file được thêm hoặc xóa khỏi directory đang hiển thị, hoặc daemon muốn giám sát file cấu hình để biết khi nào nó thay đổi.

Từ kernel 2.6.13, Linux cung cấp cơ chế **inotify**, cho phép ứng dụng giám sát sự kiện file. Cơ chế inotify thay thế cơ chế cũ hơn dnotify. Cả inotify và dnotify đều là đặc thù của Linux.

---

## 19.1 Tổng Quan

Các bước chính trong việc dùng inotify API:

1. Ứng dụng dùng `inotify_init()` để tạo inotify instance. System call này trả về file descriptor dùng để tham chiếu đến inotify instance trong các thao tác sau.
2. Ứng dụng thông báo cho kernel biết file nào quan trọng bằng cách dùng `inotify_add_watch()` để thêm watch item vào watch list của inotify instance. Mỗi watch item bao gồm pathname và bit mask liên quan. Bit mask chỉ định tập sự kiện cần giám sát. `inotify_add_watch()` trả về watch descriptor dùng để tham chiếu đến watch item trong các thao tác sau.
3. Để nhận thông báo sự kiện, ứng dụng thực hiện thao tác `read()` trên inotify file descriptor. Mỗi lần `read()` thành công trả về một hoặc nhiều structure `inotify_event`, mỗi cái chứa thông tin về sự kiện xảy ra trên một trong các pathname đang được theo dõi.
4. Khi ứng dụng hoàn tất giám sát, nó đóng inotify file descriptor, tự động xóa tất cả watch item liên quan.

Cơ chế inotify có thể giám sát file hoặc directory. Khi giám sát directory, ứng dụng được thông báo về sự kiện trên chính directory và các file bên trong.

Cơ chế giám sát inotify **không đệ quy**. Nếu ứng dụng muốn giám sát sự kiện trong toàn bộ directory subtree, nó phải gọi `inotify_add_watch()` cho từng directory trong cây.

Inotify file descriptor có thể được giám sát bằng `select()`, `poll()`, `epoll`, và signal-driven I/O.

---

## 19.2 inotify API

System call `inotify_init()` tạo inotify instance mới.

```c
#include <sys/inotify.h>
int inotify_init(void);
                                Returns file descriptor on success, or –1 on error
```

Trả về file descriptor dùng để tham chiếu đến inotify instance. Từ kernel 2.6.27, `inotify_init1()` cung cấp thêm đối số `flags`: `IN_CLOEXEC` bật close-on-exec flag, `IN_NONBLOCK` bật `O_NONBLOCK` trên file description bên dưới.

System call `inotify_add_watch()` thêm watch item mới hoặc sửa đổi watch item hiện có trong watch list của inotify instance được tham chiếu bởi `fd`.

```c
#include <sys/inotify.h>
int inotify_add_watch(int fd, const char *pathname, uint32_t mask);
                            Returns watch descriptor on success, or –1 on error
```

Đối số `pathname` xác định file cần tạo hoặc sửa đổi watch item. Caller phải có quyền read trên file này.

Đối số `mask` là bit mask chỉ định sự kiện cần giám sát.

Nếu `pathname` chưa được thêm vào watch list, `inotify_add_watch()` tạo watch item mới và trả về watch descriptor mới, không âm. Nếu `pathname` đã có trong watch list, `inotify_add_watch()` sửa đổi mask của watch item và trả về watch descriptor hiện có.

System call `inotify_rm_watch()` xóa watch item được chỉ định bởi `wd` khỏi inotify instance.

```c
#include <sys/inotify.h>
int inotify_rm_watch(int fd, uint32_t wd);
                                             Returns 0 on success, or –1 on error
```

Xóa watch gây ra sự kiện `IN_IGNORED` được tạo ra cho watch descriptor này.

---

## 19.3 inotify Event

Khi tạo hoặc sửa đổi watch bằng `inotify_add_watch()`, đối số `mask` xác định sự kiện cần giám sát.

**Bảng 19-1:** inotify event

| Giá trị bit | Vào | Ra | Mô tả |
|-------------|-----|-----|-------|
| `IN_ACCESS` | ✓ | ✓ | File được truy cập (`read()`) |
| `IN_ATTRIB` | ✓ | ✓ | Metadata file thay đổi |
| `IN_CLOSE_WRITE` | ✓ | ✓ | File mở để ghi được đóng |
| `IN_CLOSE_NOWRITE` | ✓ | ✓ | File mở read-only được đóng |
| `IN_CREATE` | ✓ | ✓ | File/directory được tạo bên trong directory được theo dõi |
| `IN_DELETE` | ✓ | ✓ | File/directory bị xóa khỏi directory được theo dõi |
| `IN_DELETE_SELF` | ✓ | ✓ | File/directory đang được theo dõi chính nó bị xóa |
| `IN_MODIFY` | ✓ | ✓ | File bị sửa đổi |
| `IN_MOVE_SELF` | ✓ | ✓ | File/directory đang được theo dõi chính nó bị di chuyển |
| `IN_MOVED_FROM` | ✓ | ✓ | File được di chuyển ra khỏi directory được theo dõi |
| `IN_MOVED_TO` | ✓ | ✓ | File được di chuyển vào directory được theo dõi |
| `IN_OPEN` | ✓ | ✓ | File được mở |
| `IN_ALL_EVENTS` | ✓ | | Shorthand cho tất cả sự kiện đầu vào trên |
| `IN_MOVE` | ✓ | | Shorthand cho `IN_MOVED_FROM\|IN_MOVED_TO` |
| `IN_CLOSE` | ✓ | | Shorthand cho `IN_CLOSE_WRITE\|IN_CLOSE_NOWRITE` |
| `IN_DONT_FOLLOW` | ✓ | | Không dereference symbolic link (từ Linux 2.6.15) |
| `IN_MASK_ADD` | ✓ | | Thêm sự kiện vào watch mask hiện tại cho pathname |
| `IN_ONESHOT` | ✓ | | Chỉ giám sát pathname cho một sự kiện |
| `IN_ONLYDIR` | ✓ | | Thất bại nếu pathname không phải directory (từ Linux 2.6.15) |
| `IN_IGNORED` | | ✓ | Watch bị xóa bởi ứng dụng hoặc kernel |
| `IN_ISDIR` | | ✓ | Filename trong trường `name` là directory |
| `IN_Q_OVERFLOW` | | ✓ | Tràn event queue |
| `IN_UNMOUNT` | | ✓ | File system chứa đối tượng bị unmount |

Một số chi tiết:
- `IN_ATTRIB`: Xảy ra khi metadata file thay đổi (quyền, ownership, link count, EA, UID, GID).
- `IN_DONT_FOLLOW`: Cho phép giám sát symbolic link thay vì file nó trỏ đến.
- `IN_MASK_ADD`: Nếu không có flag này, mask đã cho thay thế mask hiện tại. Nếu có, mask hiện tại được OR với giá trị trong `mask`.
- `IN_ONESHOT`: Giám sát pathname cho một sự kiện duy nhất; sau đó watch item tự động bị xóa.
- `IN_ONLYDIR`: Chỉ giám sát nếu pathname là directory; nếu không, thất bại với `ENOTDIR`.

---

## 19.4 Đọc inotify Event

Sau khi đăng ký watch item, ứng dụng có thể xác định sự kiện đã xảy ra bằng `read()` từ inotify file descriptor. Nếu chưa có sự kiện, `read()` block cho đến khi có (trừ khi `O_NONBLOCK` được đặt).

Sau khi có sự kiện, mỗi `read()` trả về buffer chứa một hoặc nhiều structure:

```c
struct inotify_event {
    int      wd;      /* Watch descriptor xảy ra sự kiện */
    uint32_t mask;    /* Bit mô tả sự kiện */
    uint32_t cookie;  /* Cookie cho sự kiện liên quan (cho rename()) */
    uint32_t len;     /* Kích thước trường 'name' */
    char     name[];  /* Tên file tùy chọn kết thúc null */
};
```

- **`wd`**: Cho biết watch descriptor nào sự kiện xảy ra. Hữu ích khi giám sát nhiều file/directory qua cùng inotify fd.
- **`mask`**: Bit mask mô tả sự kiện.
- **`cookie`**: Dùng để liên kết các sự kiện. Hiện tại chỉ dùng khi rename file: `IN_MOVED_FROM` và `IN_MOVED_TO` có cùng giá trị `cookie`.
- **`name`**: Khi sự kiện xảy ra trong directory được giám sát, trả về chuỗi null-terminated xác định file.
- **`len`**: Số byte được cấp phát cho trường `name` (có thể có padding).

Độ dài của mỗi inotify event là `sizeof(struct inotify_event) + len`.

Nếu buffer truyền vào `read()` quá nhỏ để chứa inotify_event tiếp theo, `read()` thất bại với `EINVAL`. Buffer nên ít nhất `(sizeof(struct inotify_event) + NAME_MAX + 1)` byte.

Các sự kiện đọc từ inotify fd tạo thành queue có thứ tự. Kernel sẽ coalesce sự kiện mới với sự kiện ở cuối queue nếu chúng có cùng `wd`, `mask`, `cookie`, và `name`.

### Chương trình ví dụ

**Listing 19-1:** Sử dụng inotify API

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––– inotify/demo_inotify.c
#include <sys/inotify.h>
#include <limits.h>
#include "tlpi_hdr.h"
static void
displayInotifyEvent(struct inotify_event *i)
{
    printf("  wd =%2d; ", i->wd);
    if (i->cookie > 0)
        printf("cookie =%4d; ", i->cookie);
    printf("mask = ");
    if (i->mask & IN_ACCESS)        printf("IN_ACCESS ");
    if (i->mask & IN_ATTRIB)        printf("IN_ATTRIB ");
    if (i->mask & IN_CLOSE_NOWRITE) printf("IN_CLOSE_NOWRITE ");
    if (i->mask & IN_CLOSE_WRITE)   printf("IN_CLOSE_WRITE ");
    if (i->mask & IN_CREATE)        printf("IN_CREATE ");
    if (i->mask & IN_DELETE)        printf("IN_DELETE ");
    if (i->mask & IN_DELETE_SELF)   printf("IN_DELETE_SELF ");
    if (i->mask & IN_IGNORED)       printf("IN_IGNORED ");
    if (i->mask & IN_ISDIR)         printf("IN_ISDIR ");
    if (i->mask & IN_MODIFY)        printf("IN_MODIFY ");
    if (i->mask & IN_MOVE_SELF)     printf("IN_MOVE_SELF ");
    if (i->mask & IN_MOVED_FROM)    printf("IN_MOVED_FROM ");
    if (i->mask & IN_MOVED_TO)      printf("IN_MOVED_TO ");
    if (i->mask & IN_OPEN)          printf("IN_OPEN ");
    if (i->mask & IN_Q_OVERFLOW)    printf("IN_Q_OVERFLOW ");
    if (i->mask & IN_UNMOUNT)       printf("IN_UNMOUNT ");
    printf("\n");
    if (i->len > 0)
        printf("  name = %s\n", i->name);
}
#define BUF_LEN (10 * (sizeof(struct inotify_event) + NAME_MAX + 1))
int
main(int argc, char *argv[])
{
    int inotifyFd, wd, j;
    char buf[BUF_LEN];
    ssize_t numRead;
    char *p;
    struct inotify_event *event;
    if (argc < 2 || strcmp(argv[1], "--help") == 0)
        usageErr("%s pathname...\n", argv[0]);
    inotifyFd = inotify_init();         /* Tạo inotify instance */
    if (inotifyFd == -1)
        errExit("inotify_init");
    for (j = 1; j < argc; j++) {
        wd = inotify_add_watch(inotifyFd, argv[j], IN_ALL_EVENTS);
        if (wd == -1)
            errExit("inotify_add_watch");
        printf("Watching %s using wd %d\n", argv[j], wd);
    }
    for (;;) {                          /* Đọc sự kiện mãi mãi */
        numRead = read(inotifyFd, buf, BUF_LEN);
        if (numRead == 0)
            fatal("read() from inotify fd returned 0!");
        if (numRead == -1)
            errExit("read");
        printf("Read %ld bytes from inotify fd\n", (long) numRead);
        for (p = buf; p < buf + numRead; ) {
            event = (struct inotify_event *) p;
            displayInotifyEvent(event);
            p += sizeof(struct inotify_event) + event->len;
        }
    }
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––– inotify/demo_inotify.c
```

Ví dụ session shell:

```
$ ./demo_inotify dir1 dir2 &
Watching dir1 using wd 1
Watching dir2 using wd 2

$ cat > dir1/aaa
Read 64 bytes from inotify fd
  wd = 1; mask = IN_CREATE
  name = aaa
  wd = 1; mask = IN_OPEN
  name = aaa

$ mv dir1/aaa dir2/bbb
Read 64 bytes from inotify fd
  wd = 1; cookie =  548; mask = IN_MOVED_FROM
  name = aaa
  wd = 2; cookie =  548; mask = IN_MOVED_TO
  name = bbb
```

Hai sự kiện chia sẻ cùng giá trị `cookie`, cho phép ứng dụng liên kết chúng.

```
$ mkdir dir2/ddd
Read 32 bytes from inotify fd
  wd = 1; mask = IN_CREATE IN_ISDIR
  name = ddd

$ rmdir dir1
Read 32 bytes from inotify fd
  wd = 1; mask = IN_DELETE_SELF
  wd = 1; mask = IN_IGNORED
```

Sự kiện `IN_IGNORED` thông báo ứng dụng rằng kernel đã xóa watch item này.

---

## 19.5 Giới Hạn Queue và File /proc

Đặt queue sự kiện inotify cần kernel memory. Kernel áp đặt các giới hạn thông qua ba file trong `/proc/sys/fs/inotify`:

- **`max_queued_events`**: Giới hạn trên về số sự kiện có thể xếp hàng trên inotify instance mới. Khi đạt giới hạn, sự kiện `IN_Q_OVERFLOW` được tạo ra và các sự kiện dư thừa bị loại bỏ. Trường `wd` của sự kiện overflow sẽ có giá trị `-1`.
- **`max_user_instances`**: Giới hạn số inotify instance có thể tạo mỗi real user ID.
- **`max_user_watches`**: Giới hạn số watch item có thể tạo mỗi real user ID.

Giá trị mặc định điển hình lần lượt là 16.384, 128, và 8.192.

---

## 19.6 Cơ Chế Cũ Giám Sát Sự Kiện File: dnotify

Linux cũng cung cấp cơ chế dnotify (từ kernel 2.4), nhưng đã bị obsolete bởi inotify. Các hạn chế của dnotify so với inotify:

- **Thông báo qua signal**: dnotify thông báo sự kiện bằng cách gửi signal cho ứng dụng — phức tạp hóa thiết kế ứng dụng và khó dùng trong library. inotify không dùng signal.
- **Đơn vị giám sát là directory**: dnotify chỉ giám sát directory, không giám sát file riêng lẻ.
- **Yêu cầu mở file descriptor**: dnotify yêu cầu ứng dụng mở file descriptor cho directory, khiến file system không thể unmount và tiêu thụ nhiều file descriptor. inotify tránh được điều này.
- **Thông tin ít chi tiết hơn**: dnotify chỉ cho biết sự kiện xảy ra trong directory nhưng không nói file nào liên quan, và cung cấp ít thông tin chi tiết về loại sự kiện.
- **Trong một số trường hợp**: dnotify không cung cấp thông báo đáng tin cậy.

---

## 19.7 Tóm Tắt

Cơ chế inotify đặc thù của Linux cho phép ứng dụng nhận thông báo khi sự kiện (file được mở, đóng, tạo, xóa, sửa đổi, đổi tên, v.v.) xảy ra trên tập file và directory được giám sát. inotify thay thế cơ chế dnotify cũ hơn.

---

## 19.8 Bài Tập

**19-1.** Viết chương trình ghi lại tất cả lần tạo file, xóa, và đổi tên dưới directory được đặt tên trong đối số command-line. Chương trình nên giám sát sự kiện trong tất cả subdirectory bên dưới directory được chỉ định. Để lấy danh sách tất cả subdirectory này, dùng `nftw()` (Mục 18.9). Khi subdirectory mới được thêm vào hoặc directory bị xóa, tập subdirectory được giám sát phải được cập nhật tương ứng.
