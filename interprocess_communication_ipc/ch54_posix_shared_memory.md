## Chương 54
# POSIX SHARED MEMORY

Trong các chương trước, chúng ta đã xem xét hai kỹ thuật cho phép các process không liên quan chia sẻ các vùng bộ nhớ để thực hiện IPC: System V shared memory (Chương 48) và shared file mappings (Mục 49.4.2). Cả hai kỹ thuật này đều có những nhược điểm tiềm ẩn:

- Mô hình System V shared memory, sử dụng key và identifier, không nhất quán với mô hình I/O UNIX tiêu chuẩn, sử dụng tên file và descriptor. Sự khác biệt này có nghĩa là chúng ta cần một bộ system call và lệnh hoàn toàn mới để làm việc với các System V shared memory segment.
- Sử dụng shared file mapping cho IPC đòi hỏi phải tạo một file trên đĩa, ngay cả khi chúng ta không quan tâm đến việc có một backing store lâu dài cho vùng được chia sẻ. Ngoài sự bất tiện khi phải tạo file, kỹ thuật này phát sinh một số file I/O overhead.

Do những nhược điểm này, POSIX.1b đã định nghĩa một API shared memory mới: POSIX shared memory, là chủ đề của chương này.

> POSIX nói về các shared memory object, trong khi System V nói về các shared memory segment. Những sự khác biệt về thuật ngữ này là do lịch sử — cả hai thuật ngữ đều được sử dụng để chỉ các vùng bộ nhớ được chia sẻ giữa các process.

## **54.1 Tổng quan**

POSIX shared memory cho phép chúng ta chia sẻ một vùng được ánh xạ giữa các process không liên quan mà không cần tạo một file ánh xạ tương ứng. POSIX shared memory được hỗ trợ trên Linux kể từ kernel 2.4.

SUSv3 không quy định bất kỳ chi tiết nào về cách hiện thực POSIX shared memory. Đặc biệt, không có yêu cầu sử dụng hệ thống file (thực hay ảo) để xác định các shared memory object, mặc dù nhiều hiện thực UNIX sử dụng hệ thống file cho mục đích này. Một số hiện thực UNIX tạo tên cho các shared memory object dưới dạng file ở một vị trí đặc biệt trong hệ thống file tiêu chuẩn. Linux sử dụng một hệ thống file tmpfs chuyên dụng (Mục 14.10) được mount dưới thư mục `/dev/shm`. Hệ thống file này có kernel persistence — các shared memory object mà nó chứa sẽ tồn tại ngay cả khi không có process nào đang mở chúng, nhưng chúng sẽ bị mất nếu hệ thống bị tắt.

> Tổng lượng bộ nhớ trong tất cả các vùng POSIX shared memory trên hệ thống bị giới hạn bởi kích thước của hệ thống file tmpfs cơ sở. Hệ thống file này thường được mount lúc khởi động với một số kích thước mặc định (ví dụ: 256 MB). Nếu cần, superuser có thể thay đổi kích thước của hệ thống file bằng cách remount nó bằng lệnh `mount –o remount,size=<num-bytes>`.

Để sử dụng một POSIX shared memory object, chúng ta thực hiện hai bước:

1. Sử dụng hàm `shm_open()` để mở một object với tên được chỉ định. (Chúng ta đã mô tả các quy tắc chi phối việc đặt tên cho các POSIX shared memory object trong Mục 51.1.) Hàm `shm_open()` tương tự với system call `open()`. Nó tạo một shared memory object mới hoặc mở một object hiện có. Là kết quả hàm của nó, `shm_open()` trả về một file descriptor tham chiếu đến object.
2. Truyền file descriptor thu được ở bước trước vào một lời gọi `mmap()` chỉ định `MAP_SHARED` trong đối số `flags`. Điều này ánh xạ shared memory object vào không gian địa chỉ ảo của process. Như với các cách sử dụng khác của `mmap()`, sau khi chúng ta đã ánh xạ object, chúng ta có thể đóng file descriptor mà không ảnh hưởng đến mapping. Tuy nhiên, chúng ta có thể cần giữ file descriptor mở để sử dụng tiếp theo trong các lời gọi đến `fstat()` và `ftruncate()` (xem Mục 54.2).

Mối quan hệ giữa `shm_open()` và `mmap()` cho POSIX shared memory tương tự như giữa `shmget()` và `shmat()` cho System V shared memory. Nguồn gốc của quy trình hai bước (`shm_open()` cộng `mmap()`) để sử dụng các POSIX shared memory object thay vì sử dụng một hàm duy nhất thực hiện cả hai tác vụ là do lịch sử. Khi ủy ban POSIX thêm tính năng này, lời gọi `mmap()` đã tồn tại ([Stevens, 1999]). Thực tế, tất cả những gì chúng ta đang làm là thay thế các lời gọi `open()` bằng các lời gọi `shm_open()`, với sự khác biệt là sử dụng `shm_open()` không yêu cầu tạo file trong hệ thống file trên đĩa.

Vì một shared memory object được tham chiếu bằng cách sử dụng một file descriptor, chúng ta có thể sử dụng hữu ích nhiều file descriptor system call đã được định nghĩa trong hệ thống UNIX (ví dụ: `ftruncate()`), thay vì cần các system call mới có mục đích đặc biệt (như được yêu cầu cho System V shared memory).

## **54.2 Tạo Shared Memory Object**

Hàm `shm_open()` tạo và mở một shared memory object mới hoặc mở một object hiện có. Các đối số cho `shm_open()` tương tự như các đối số cho `open()`.

```
#include <fcntl.h> /* Defines O_* constants */
#include <sys/stat.h> /* Defines mode constants */
#include <sys/mman.h>
int shm_open(const char *name, int oflag, mode_t mode);
                             Returns file descriptor on success, or –1 on error
```

Đối số `name` xác định shared memory object cần được tạo hoặc mở. Đối số `oflag` là một mask các bit thay đổi hành vi của lời gọi. Các giá trị có thể được bao gồm trong mask này được tóm tắt trong Bảng 54-1.

| Bảng 54-1: Các giá trị bit cho đối số `oflag` của `shm_open()` |  |
|---------------------------------------------------------------|--|

| Flag     | Mô tả                                                    |
|----------|----------------------------------------------------------|
| `O_CREAT`  | Tạo object nếu nó chưa tồn tại                         |
| `O_EXCL`   | Với `O_CREAT`, tạo object độc quyền                    |
| `O_RDONLY` | Mở để truy cập chỉ đọc                                 |
| `O_RDWR`   | Mở để truy cập đọc-ghi                                 |
| `O_TRUNC`  | Cắt ngắn object về độ dài bằng không                  |

Một trong những mục đích của đối số `oflag` là xác định chúng ta đang mở một shared memory object hiện có hay tạo và mở một object mới. Nếu `oflag` không bao gồm `O_CREAT`, chúng ta đang mở một object hiện có. Nếu `O_CREAT` được chỉ định, thì object được tạo nếu nó chưa tồn tại. Việc chỉ định `O_EXCL` kết hợp với `O_CREAT` là một yêu cầu để đảm bảo rằng người gọi là người tạo object; nếu object đã tồn tại, một lỗi sẽ xảy ra (`EEXIST`).

Đối số `oflag` cũng cho biết loại truy cập mà process đang gọi sẽ thực hiện đối với shared memory object, bằng cách chỉ định chính xác một trong các giá trị `O_RDONLY` hoặc `O_RDWR`.

Giá trị flag còn lại, `O_TRUNC`, khiến một lần mở thành công của một shared memory object hiện có cắt ngắn object về độ dài bằng không.

> Trên Linux, việc cắt ngắn xảy ra ngay cả khi mở chỉ đọc. Tuy nhiên, SUSv3 nói rằng kết quả của việc sử dụng `O_TRUNC` với một lần mở chỉ đọc là không xác định, vì vậy chúng ta không thể dựa vào một hành vi cụ thể trong trường hợp này một cách portable.

Khi một shared memory object mới được tạo, quyền sở hữu của nó được lấy từ các user ID và group ID hiệu quả của process đang gọi `shm_open()`, và các quyền của object được đặt theo giá trị được cung cấp trong đối số bitmask `mode`. Các giá trị bit cho `mode` giống như đối với file (Bảng 15-4, trang 295). Như với system call `open()`, permissions mask trong `mode` được mask với process umask (Mục 15.4.6). Không giống như `open()`, đối số `mode` luôn được yêu cầu cho một lời gọi đến `shm_open()`; nếu chúng ta không tạo một object mới, đối số này nên được chỉ định là 0.

Flag close-on-exec (`FD_CLOEXEC`, Mục 27.4) được đặt trên file descriptor được trả về bởi `shm_open()`, để file descriptor được tự động đóng nếu process thực hiện `exec()`. (Điều này nhất quán với thực tế là các mapping được hủy ánh xạ khi `exec()` được thực hiện.)

Khi một shared memory object mới được tạo, ban đầu nó có độ dài bằng không. Điều này có nghĩa là, sau khi tạo một shared memory object mới, chúng ta thường gọi `ftruncate()` (Mục 5.8) để đặt kích thước của object trước khi gọi `mmap()`. Sau lời gọi `mmap()`, chúng ta cũng có thể sử dụng `ftruncate()` để mở rộng hoặc thu nhỏ shared memory object như mong muốn, lưu ý các điểm được thảo luận trong Mục 49.4.3.

Khi một shared memory object được mở rộng, các byte mới được thêm vào được tự động khởi tạo thành 0.

Tại bất kỳ thời điểm nào, chúng ta có thể áp dụng `fstat()` (Mục 15.1) cho file descriptor được trả về bởi `shm_open()` để thu được một cấu trúc `stat` có các trường chứa thông tin về shared memory object, bao gồm kích thước của nó (`st_size`), các quyền (`st_mode`), owner (`st_uid`), và group (`st_gid`). (Đây là các trường duy nhất mà SUSv3 yêu cầu `fstat()` đặt trong cấu trúc `stat`, mặc dù Linux cũng trả về thông tin có ý nghĩa trong các trường thời gian, cũng như nhiều thông tin ít hữu ích hơn khác trong các trường còn lại.)

Các quyền và quyền sở hữu của một shared memory object có thể được thay đổi bằng cách sử dụng `fchmod()` và `fchown()`, tương ứng.

### **Chương trình ví dụ**

Listing 54-1 cung cấp một ví dụ đơn giản về việc sử dụng `shm_open()`, `ftruncate()`, và `mmap()`. Chương trình này tạo một shared memory object có kích thước được chỉ định bởi một đối số dòng lệnh, và ánh xạ object vào không gian địa chỉ ảo của process. (Bước ánh xạ là dư thừa, vì chúng ta thực sự không làm gì với shared memory, nhưng nó giúp minh họa việc sử dụng `mmap()`.) Chương trình cho phép sử dụng các tùy chọn dòng lệnh để chọn các flag (`O_CREAT` và `O_EXCL`) cho lời gọi `shm_open()`.

Trong ví dụ sau, chúng ta sử dụng chương trình này để tạo một shared memory object 10.000 byte, sau đó sử dụng `ls` để hiển thị object này trong `/dev/shm`:

```
$ ./pshm_create -c /demo_shm 10000
$ ls -l /dev/shm
total 0
-rw------- 1 mtk users 10000 Jun 20 11:31 demo_shm
```

**Listing 54-1:** Tạo một POSIX shared memory object

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pshm/pshm_create.c
#include <sys/stat.h>
#include <fcntl.h>
#include <sys/mman.h>
#include "tlpi_hdr.h"
```

```
static void
usageError(const char *progName)
{
 fprintf(stderr, "Usage: %s [-cx] name size [octal-perms]\n", progName);
 fprintf(stderr, " -c Create shared memory (O_CREAT)\n");
 fprintf(stderr, " -x Create exclusively (O_EXCL)\n");
 exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
 int flags, opt, fd;
 mode_t perms;
 size_t size;
 void *addr;
 flags = O_RDWR;
 while ((opt = getopt(argc, argv, "cx")) != -1) {
 switch (opt) {
 case 'c': flags |= O_CREAT; break;
 case 'x': flags |= O_EXCL; break;
 default: usageError(argv[0]);
 }
 }
 if (optind + 1 >= argc)
 usageError(argv[0]);
 size = getLong(argv[optind + 1], GN_ANY_BASE, "size");
 perms = (argc <= optind + 2) ? (S_IRUSR | S_IWUSR) :
 getLong(argv[optind + 2], GN_BASE_8, "octal-perms");
 /* Create shared memory object and set its size */
 fd = shm_open(argv[optind], flags, perms);
 if (fd == -1)
 errExit("shm_open");
 if (ftruncate(fd, size) == -1)
 errExit("ftruncate");
 /* Map shared memory object */
 addr = mmap(NULL, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
 if (addr == MAP_FAILED)
 errExit("mmap");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pshm/pshm_create.c
```

## **54.3 Sử Dụng Shared Memory Object**

Listing 54-2 và Listing 54-3 minh họa việc sử dụng một shared memory object để truyền dữ liệu từ một process sang process khác. Chương trình trong Listing 54-2 sao chép chuỗi có trong đối số dòng lệnh thứ hai của nó vào shared memory object hiện có được đặt tên trong đối số dòng lệnh đầu tiên của nó. Trước khi ánh xạ object và thực hiện sao chép, chương trình sử dụng `ftruncate()` để thay đổi kích thước shared memory object sao cho có cùng độ dài với chuỗi sẽ được sao chép.

**Listing 54-2:** Sao chép dữ liệu vào một POSIX shared memory object

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– pshm/pshm_write.c
#include <fcntl.h>
#include <sys/mman.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 int fd;
 size_t len; /* Size of shared memory object */
 char *addr;
 if (argc != 3 || strcmp(argv[1], "--help") == 0)
 usageErr("%s shm-name string\n", argv[0]);
 fd = shm_open(argv[1], O_RDWR, 0); /* Open existing object */
 if (fd == -1)
 errExit("shm_open");
 len = strlen(argv[2]);
 if (ftruncate(fd, len) == -1) /* Resize object to hold string */
 errExit("ftruncate");
 printf("Resized to %ld bytes\n", (long) len);
 addr = mmap(NULL, len, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
 if (addr == MAP_FAILED)
 errExit("mmap");
 if (close(fd) == -1)
 errExit("close"); /* 'fd' is no longer needed */
 printf("copying %ld bytes\n", (long) len);
 memcpy(addr, argv[2], len); /* Copy string to shared memory */
   exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– pshm/pshm_write.c
```

Chương trình trong Listing 54-3 hiển thị chuỗi trong shared memory object hiện có được đặt tên trong đối số dòng lệnh của nó trên standard output. Sau khi gọi `shm_open()`, chương trình sử dụng `fstat()` để xác định kích thước của shared memory và sử dụng kích thước đó trong lời gọi `mmap()` để ánh xạ object và trong lời gọi `write()` để in chuỗi.

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– pshm/pshm_read.c
#include <fcntl.h>
#include <sys/mman.h>
#include <sys/stat.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 int fd;
 char *addr;
 struct stat sb;
 if (argc != 2 || strcmp(argv[1], "--help") == 0)
 usageErr("%s shm-name\n", argv[0]);
 fd = shm_open(argv[1], O_RDONLY, 0); /* Open existing object */
 if (fd == -1)
 errExit("shm_open");
 /* Use shared memory object size as length argument for mmap()
 and as number of bytes to write() */
 if (fstat(fd, &sb) == -1)
 errExit("fstat");
 addr = mmap(NULL, sb.st_size, PROT_READ, MAP_SHARED, fd, 0);
 if (addr == MAP_FAILED)
 errExit("mmap");
 if (close(fd) == -1); /* 'fd' is no longer needed */
 errExit("close");
 write(STDOUT_FILENO, addr, sb.st_size);
 printf("\n");
   exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– pshm/pshm_read.c
```

Phiên shell sau đây minh họa việc sử dụng các chương trình trong Listing 54-2 và Listing 54-3. Đầu tiên chúng ta tạo một shared memory object có độ dài bằng không bằng cách sử dụng chương trình trong Listing 54-1.

```
$ ./pshm_create -c /demo_shm 0
$ ls -l /dev/shm Check the size of object
total 4
-rw------- 1 mtk users 0 Jun 21 13:33 demo_shm
```

Sau đó chúng ta sử dụng chương trình trong Listing 54-2 để sao chép một chuỗi vào shared memory object:

```
$ ./pshm_write /demo_shm 'hello'
$ ls -l /dev/shm Check that object has changed in size
total 4
-rw------- 1 mtk users 5 Jun 21 13:33 demo_shm
```

Từ đầu ra, chúng ta có thể thấy rằng chương trình đã thay đổi kích thước shared memory object sao cho đủ lớn để chứa chuỗi được chỉ định.

Cuối cùng, chúng ta sử dụng chương trình trong Listing 54-3 để hiển thị chuỗi trong shared memory object:

```
$ ./pshm_read /demo_shm
hello
```

Các ứng dụng thường phải sử dụng một kỹ thuật đồng bộ hóa nào đó để cho phép các process điều phối quyền truy cập vào shared memory của chúng. Trong ví dụ phiên shell được hiển thị ở đây, sự điều phối được cung cấp bởi người dùng chạy các chương trình lần lượt. Thông thường, các ứng dụng thay vào đó sẽ sử dụng một synchronization primitive (ví dụ: semaphores) để điều phối quyền truy cập vào một shared memory object.

## **54.4 Xóa Shared Memory Object**

SUSv3 yêu cầu các POSIX shared memory object có ít nhất kernel persistence; tức là, chúng tiếp tục tồn tại cho đến khi chúng được xóa rõ ràng hoặc hệ thống được khởi động lại. Khi một shared memory object không còn cần thiết, nó nên được xóa bằng cách sử dụng `shm_unlink()`.

```
#include <sys/mman.h>
int shm_unlink(const char *name);
                                            Returns 0 on success, or –1 on error
```

Hàm `shm_unlink()` xóa shared memory object được chỉ định bởi `name`. Việc xóa một shared memory object không ảnh hưởng đến các mapping hiện có của object (những mapping này sẽ vẫn còn hiệu lực cho đến khi các process tương ứng gọi `munmap()` hoặc kết thúc), nhưng ngăn các lời gọi `shm_open()` tiếp theo mở object. Khi tất cả các process đã hủy ánh xạ object, object bị xóa và nội dung của nó bị mất.

Chương trình trong Listing 54-4 sử dụng `shm_unlink()` để xóa shared memory object được chỉ định trong đối số dòng lệnh của chương trình.

**Listing 54-4:** Sử dụng `shm_unlink()` để unlink một POSIX shared memory object

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pshm/pshm_unlink.c
#include <fcntl.h>
#include <sys/mman.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 if (argc != 2 || strcmp(argv[1], "--help") == 0)
 usageErr("%s shm-name\n", argv[0]);
   if (shm_unlink(argv[1]) == -1)
 errExit("shm_unlink");
   exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– pshm/pshm_unlink.c
```

## **54.5 So Sánh Giữa Các API Shared Memory**

Đến nay, chúng ta đã xem xét một số kỹ thuật khác nhau để chia sẻ các vùng bộ nhớ giữa các process không liên quan:

- System V shared memory (Chương 48);
- shared file mappings (Mục 49.4.2); và
- POSIX shared memory object (chủ đề của chương này).

Nhiều điểm chúng ta đề cập trong phần này cũng liên quan đến shared anonymous mappings (Mục 49.7), được sử dụng để chia sẻ bộ nhớ giữa các process có liên quan thông qua `fork()`.

Một số điểm áp dụng cho tất cả các kỹ thuật này:

- Chúng cung cấp IPC nhanh, và các ứng dụng thường phải sử dụng một semaphore (hoặc synchronization primitive khác) để đồng bộ hóa quyền truy cập vào vùng được chia sẻ.
- Sau khi vùng shared memory đã được ánh xạ vào không gian địa chỉ ảo của process, nó trông giống như bất kỳ phần nào khác của không gian bộ nhớ của process.
- Hệ thống đặt các vùng shared memory trong không gian địa chỉ ảo của process theo cách tương tự nhau. Chúng ta đã phác thảo vị trí này khi mô tả System V shared memory trong Mục 48.5. File `/proc/PID/maps` dành riêng cho Linux liệt kê thông tin về tất cả các loại vùng shared memory.
- Giả sử rằng chúng ta không cố ánh xạ một vùng shared memory tại một địa chỉ cố định, chúng ta nên đảm bảo rằng tất cả các tham chiếu đến các vị trí trong vùng được tính dưới dạng offset (thay vì pointer), vì vùng có thể được đặt tại các địa chỉ ảo khác nhau trong các process khác nhau (Mục 48.6).
- Các hàm được mô tả trong Chương 50 hoạt động trên các vùng virtual memory có thể được áp dụng cho các vùng shared memory được tạo bằng bất kỳ kỹ thuật nào trong số này.

Cũng có một số sự khác biệt đáng chú ý giữa các kỹ thuật cho shared memory:

- Thực tế là nội dung của một shared file mapping được đồng bộ hóa với file được ánh xạ cơ sở có nghĩa là dữ liệu được lưu trữ trong một vùng shared memory có thể tồn tại sau khi hệ thống khởi động lại.
- System V và POSIX shared memory sử dụng các cơ chế khác nhau để xác định và tham chiếu đến một shared memory object. System V sử dụng sơ đồ riêng của mình với key và identifier, không phù hợp với mô hình I/O UNIX tiêu chuẩn và đòi hỏi các system call riêng biệt (ví dụ: `shmctl()`) và lệnh (`ipcs` và `ipcrm`). Ngược lại, POSIX shared memory sử dụng tên và file descriptor, và do đó các shared memory object có thể được kiểm tra và thao tác bằng cách sử dụng nhiều system call UNIX hiện có (ví dụ: `fstat()` và `fchmod()`).
- Kích thước của một System V shared memory segment được cố định tại thời điểm tạo (thông qua `shmget()`). Ngược lại, đối với một mapping được hỗ trợ bởi một file hoặc bởi một POSIX shared memory object, chúng ta có thể sử dụng `ftruncate()` để điều chỉnh kích thước của object cơ sở, và sau đó tạo lại mapping bằng cách sử dụng `munmap()` và `mmap()` (hoặc `mremap()` dành riêng cho Linux).

 Về mặt lịch sử, System V shared memory được phổ biến rộng rãi hơn `mmap()` và POSIX shared memory, mặc dù hầu hết các hiện thực UNIX bây giờ đều cung cấp tất cả các kỹ thuật này.

Với ngoại lệ của điểm cuối về tính portable, các sự khác biệt được liệt kê ở trên là lợi thế có lợi cho shared file mappings và POSIX shared memory object. Do đó, trong các ứng dụng mới, một trong những giao diện này có thể được ưa thích hơn System V shared memory. Cái nào chúng ta chọn tùy thuộc vào việc chúng ta có cần một backing store lâu dài hay không. Shared file mappings cung cấp backing store như vậy; POSIX shared memory object cho phép chúng ta tránh overhead của việc sử dụng file trên đĩa khi không cần backing store.

## **54.6 Tóm Tắt**

Một POSIX shared memory object được sử dụng để chia sẻ một vùng bộ nhớ giữa các process không liên quan mà không cần tạo file trên đĩa cơ sở. Để làm điều này, chúng ta thay thế lời gọi `open()` thường đứng trước `mmap()` bằng lời gọi `shm_open()`. Lời gọi `shm_open()` tạo một file trong một hệ thống file dựa trên bộ nhớ, và chúng ta có thể sử dụng các file descriptor system call truyền thống để thực hiện các thao tác khác nhau trên file ảo này. Đặc biệt, `ftruncate()` phải được sử dụng để đặt kích thước của shared memory object, vì ban đầu nó có độ dài bằng không.

Chúng ta đã mô tả ba kỹ thuật để chia sẻ các vùng bộ nhớ giữa các process không liên quan: System V shared memory, shared file mappings, và POSIX shared memory object. Có một số điểm tương đồng giữa ba kỹ thuật. Cũng có một số sự khác biệt quan trọng, và, ngoại trừ vấn đề về tính portable, những sự khác biệt này có lợi cho shared file mappings và POSIX shared memory object.

## **54.7 Bài Tập**

**54-1.** Viết lại các chương trình trong Listing 48-2 (`svshm_xfr_writer.c`) và Listing 48-3 (`svshm_xfr_reader.c`) để sử dụng POSIX shared memory object thay vì System V shared memory.
