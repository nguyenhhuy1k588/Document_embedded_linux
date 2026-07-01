## Chương 4
# **FILE I/O: MÔ HÌNH I/O PHỔ QUÁT**

Bây giờ chúng ta bắt đầu nghiên cứu một cách trang trọng API system call. File là một nơi tốt để bắt đầu, vì chúng là trung tâm của triết lý UNIX. Tiêu điểm của chương này là các system call được sử dụng để thực hiện file input và output.

Chúng ta giới thiệu khái niệm file descriptor, sau đó xem xét các system call tạo nên cái gọi là mô hình I/O phổ quát. Đây là các system call mở và đóng file, và đọc và ghi dữ liệu.

Chúng ta tập trung vào I/O trên disk file. Tuy nhiên, phần lớn tài liệu được đề cập ở đây là liên quan đến các chương sau, vì các system call tương tự được sử dụng để thực hiện I/O trên tất cả các loại file, chẳng hạn như pipes và terminals.

Chapter [5](#page-68-0) mở rộng thảo luận trong chương này với các chi tiết bổ sung về file I/O. Một khía cạnh khác của file I/O, buffering, phức tạp đủ để xứng đáng có chương riêng. Chapter 13 trình bày I/O buffering trong kernel và trong stdio library.

## **4.1 Tổng quan**

Tất cả các system call để thực hiện I/O đều tham chiếu đến các file mở bằng file descriptor, một số nguyên không âm (thường là nhỏ). File descriptor được sử dụng để tham chiếu đến tất cả các loại file mở, bao gồm pipes, FIFOs, sockets, terminals, devices, và regular file. Mỗi process có bộ file descriptor riêng của nó.

Theo quy ước, hầu hết các chương trình mong muốn có thể sử dụng ba standard file descriptor được liệt kê trong [Table 4-1](#page-49-0). Ba descriptor này được mở thay mặt cho chương trình bởi shell, trước khi chương trình được khởi động. Hay nói chính xác hơn, chương trình kế thừa các bản sao của file descriptor của shell, và shell thường hoạt động với ba file descriptor này luôn mở. (Trong shell tương tác, ba file descriptor này thường tham chiếu đến terminal mà dưới đó shell đang chạy.) Nếu I/O redirections được chỉ định trên dòng lệnh, thì shell đảm bảo rằng file descriptor được sửa đổi một cách thích hợp trước khi khởi động chương trình.

<span id="page-49-0"></span>**Table 4-1:** Standard file descriptor

| File descriptor | Mục đích         | POSIX name    | stdio stream |
|-----------------|------------------|---------------|--------------|
| 0               | standard input   | STDIN_FILENO  | stdin        |
| 1               | standard output  | STDOUT_FILENO | stdout       |
| 2               | standard error   | STDERR_FILENO | stderr       |

Khi tham chiếu đến những file descriptor này trong một chương trình, chúng ta có thể sử dụng các số (0, 1, hoặc 2) hoặc, tốt hơn là, các tên tiêu chuẩn POSIX được định nghĩa trong <unistd.h>.

> Mặc dù các biến `stdin`, `stdout`, và `stderr` ban đầu tham chiếu đến standard input, output, và error của process, chúng có thể được thay đổi để tham chiếu đến bất kỳ file nào bằng cách sử dụng hàm library `freopen()`. Như một phần của hoạt động của nó, `freopen()` có thể thay đổi file descriptor bên dưới stream được mở lại. Nói cách khác, sau khi `freopen()` trên `stdout`, chẳng hạn, không còn an toàn nữa để giả định rằng file descriptor bên dưới vẫn còn là 1.

Các system call chính sau đây là bốn system call chính để thực hiện file I/O (các ngôn ngữ lập trình và gói phần mềm thường sử dụng các lệnh gọi này chỉ gián tiếp, thông qua I/O library):

-  `fd = open(pathname, flags, mode)` mở file được xác định bởi pathname, trả về một file descriptor được sử dụng để tham chiếu đến file mở trong các lệnh gọi tiếp theo. Nếu file không tồn tại, `open()` có thể tạo nó, tùy thuộc vào cài đặt của đối số bitmask flags. Đối số flags cũng chỉ định liệu file sẽ được mở để đọc, ghi, hay cả hai. Đối số mode chỉ định quyền hạn sẽ được đặt trên file nếu nó được tạo bởi lệnh gọi này. Nếu lệnh gọi `open()` không được sử dụng để tạo file, đối số này bị bỏ qua và có thể bị bỏ sót.
-  `numread = read(fd, buffer, count)` đọc tối đa count byte từ file mở được tham chiếu bởi fd và lưu chúng vào buffer. Lệnh gọi `read()` trả về số byte thực tế được đọc. Nếu không thể đọc thêm byte nào (tức là, gặp end-of-file), `read()` trả về 0.
-  `numwritten = write(fd, buffer, count)` ghi tối đa count byte từ buffer vào file mở được tham chiếu bởi fd. Lệnh gọi `write()` trả về số byte thực tế được ghi, có thể nhỏ hơn count.
-  `status = close(fd)` được gọi sau khi tất cả I/O đã hoàn tất, để giải phóng file descriptor fd và các kernel resource liên quan của nó.

Trước khi chúng ta đi vào chi tiết của các system call này, chúng ta cung cấp một bản demo ngắn về cách sử dụng của chúng trong [Listing 4-1.](#page-50-0) Chương trình này là một phiên bản đơn giản của lệnh cp(1). Nó sao chép nội dung của file hiện tại được đặt tên trong đối số dòng lệnh đầu tiên của nó vào file mới được đặt tên trong đối số dòng lệnh thứ hai.

Chúng ta có thể sử dụng chương trình trong [Listing 4-1](#page-50-0) như sau:

#### \$ **./copy oldfile newfile**

#### <span id="page-50-0"></span>**Listing 4-1:** Sử dụng I/O system call

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– fileio/copy.c
#include <sys/stat.h>
#include <fcntl.h>
#include "tlpi_hdr.h"
#ifndef BUF_SIZE /* Allow "cc -D" to override definition */
#define BUF_SIZE 1024
#endif
int
main(int argc, char *argv[])
{
 int inputFd, outputFd, openFlags;
 mode_t filePerms;
 ssize_t numRead;
 char buf[BUF_SIZE];
 if (argc != 3 || strcmp(argv[1], "--help") == 0)
 usageErr("%s old-file new-file\n", argv[0]);
 /* Open input and output files */
 inputFd = open(argv[1], O_RDONLY);
 if (inputFd == -1)
 errExit("opening file %s", argv[1]);
 openFlags = O_CREAT | O_WRONLY | O_TRUNC;
 filePerms = S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP |
 S_IROTH | S_IWOTH; /* rw-rw-rw- */
 outputFd = open(argv[2], openFlags, filePerms);
 if (outputFd == -1)
 errExit("opening file %s", argv[2]);
 /* Transfer data until we encounter end of input or an error */
 while ((numRead = read(inputFd, buf, BUF_SIZE)) > 0)
 if (write(outputFd, buf, numRead) != numRead)
 fatal("couldn't write whole buffer");
 if (numRead == -1)
 errExit("read");
 if (close(inputFd) == -1)
 errExit("close input");
 if (close(outputFd) == -1)
 errExit("close output");
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– fileio/copy.c
```

## **4.2 Tính phổ quát của I/O**

Một trong những đặc điểm nổi bật của mô hình I/O UNIX là khái niệm tính phổ quát của I/O. Điều này có nghĩa là bốn system call tương tự—`open()`, `read()`, `write()`, và `close()`—được sử dụng để thực hiện I/O trên tất cả các loại file, bao gồm các device như terminals. Do đó, nếu chúng ta viết một chương trình chỉ sử dụng các system call này, chương trình đó sẽ hoạt động trên bất kỳ loại file nào. Ví dụ, các cách sử dụng hợp lệ sau của chương trình trong [Listing 4-1](#page-50-0):

```
$ ./copy test test.old Copy a regular file
$ ./copy a.txt /dev/tty Copy a regular file to this terminal
$ ./copy /dev/tty b.txt Copy input from this terminal to a regular file
$ ./copy /dev/pts/16 /dev/tty Copy input from another terminal
```

Tính phổ quát của I/O được đạt được bằng cách đảm bảo rằng mỗi file system và device driver triển khai bộ system call I/O tương tự. Vì các chi tiết cụ thể cho file system hoặc device được xử lý trong kernel, chúng ta có thể bỏ qua các yếu tố cụ thể của device khi viết các chương trình ứng dụng. Khi cần truy cập vào các tính năng cụ thể của một file system hoặc device, chương trình có thể sử dụng system call `ioctl()` để tất cả [\(Section 4.8\)](#page-65-0), cung cấp một giao diện cho các tính năng ngoài mô hình I/O phổ quát.

# **4.3 Mở một File: open()**

System call `open()` mở một file hiện tại hoặc tạo và mở một file mới.

```
#include <sys/stat.h>
#include <fcntl.h>
int open(const char *pathname, int flags, ... /* mode_t mode */);
                               Returns file descriptor on success, or –1 on error
```

File cần được mở được xác định bởi đối số pathname. Nếu pathname là một symbolic link, nó sẽ được dereference. Khi thành công, `open()` trả về một file descriptor được sử dụng để tham chiếu đến file trong các system call tiếp theo. Nếu xảy ra lỗi, `open()` trả về –1 và `errno` được đặt tương ứng.

Đối số flags là một bit mask chỉ định chế độ truy cập của file, sử dụng một trong các hằng số được hiển thị trong [Table 4-2.](#page-52-0)

> Các triển khai UNIX sớm sử dụng các số 0, 1, và 2 thay vì các tên được hiển thị trong Table [4-2](#page-52-0). Hầu hết các triển khai UNIX hiện đại định nghĩa các hằng số này có những giá trị đó. Do đó, chúng ta có thể thấy rằng `O_RDWR` không tương đương với `O_RDONLY | O_WRONLY`; sự kết hợp sau là một lỗi logic.

Khi `open()` được sử dụng để tạo một file mới, đối số bit-mask mode chỉ định quyền hạn sẽ được đặt trên file. (Kiểu dữ liệu `mode_t` được sử dụng để gõ mode là một kiểu số nguyên được chỉ định trong SUSv3.) Nếu lệnh gọi `open()` không chỉ định `O_CREAT`, mode có thể bị bỏ sót.

<span id="page-52-0"></span>**Table 4-2:** File access modes

| Access mode | Mô tả                              |
|-------------|-----------------------------------|
| O_RDONLY    | Mở file để chỉ đọc                |
| O_WRONLY    | Mở file để chỉ ghi                |
| O_RDWR      | Mở file để đọc và ghi             |

Chúng ta mô tả chi tiết quyền file trong Section 15.4. Sau này, chúng ta sẽ thấy rằng quyền thực tế được đặt trên một file mới không chỉ phụ thuộc vào đối số mode, mà còn phụ thuộc vào process umask (Section 15.4.6) và danh sách kiểm soát truy cập mặc định (tùy chọn) của thư mục cha (Section 17.6). Trong khi chờ đợi, chúng ta sẽ lưu ý rằng đối số mode có thể được chỉ định dưới dạng một số (thường ở dạng octal) hoặc, tốt hơn là, bằng cách ORing (|) với nhau không hoặc nhiều hằng số bit-mask được liệt kê trong Table 15-4, trên trang 295.

[Listing 4-2](#page-52-1) cho thấy các ví dụ về cách sử dụng `open()`, một số trong số đó sử dụng các bit flags bổ sung mà chúng ta mô tả ngắn gọn.

<span id="page-52-1"></span>**Listing 4-2:** Ví dụ về cách sử dụng open()

```
 /* Open existing file for reading */
 fd = open("startup", O_RDONLY);
 if (fd == -1)
 errExit("open");
   /* Open new or existing file for reading and writing, truncating to zero 
      bytes; file permissions read+write for owner, nothing for all others */
 fd = open("myfile", O_RDWR | O_CREAT | O_TRUNC, S_IRUSR | S_IWUSR);
 if (fd == -1)
 errExit("open");
 /* Open new or existing file for writing; writes should always
 append to end of file */
 fd = open("w.log", O_WRONLY | O_CREAT | O_TRUNC | O_APPEND,
 S_IRUSR | S_IWUSR);
 if (fd == -1)
 errExit("open");
```

#### **Số file descriptor được trả về bởi open()**

SUSv3 chỉ định rằng nếu `open()` thành công, nó được đảm bảo sử dụng file descriptor chưa sử dụng có số thấp nhất cho process. Chúng ta có thể sử dụng tính năng này để đảm bảo rằng một file được mở bằng một file descriptor cụ thể. Ví dụ, trình tự sau đây đảm bảo rằng một file được mở bằng standard input (file descriptor 0).

```
if (close(STDIN_FILENO) == -1) /* Close file descriptor 0 */
 errExit("close");
fd = open(pathname, O_RDONLY);
if (fd == -1)
 errExit("open");
```

Vì file descriptor 0 không được sử dụng, `open()` được đảm bảo sẽ mở file bằng descriptor đó. Trong [Section 5.5](#page-75-0), chúng ta xem xét việc sử dụng `dup2()` và `fcntl()` để đạt được kết quả tương tự, nhưng với kiểm soát linh hoạt hơn đối với file descriptor được sử dụng. Trong phần đó, chúng ta cũng hiển thị một ví dụ về lý do tại sao có thể hữu ích để kiểm soát file descriptor mà trên đó một file được mở.

## **4.3.1 Đối số flags của open()**

<span id="page-53-0"></span>Trong một số lệnh gọi `open()` được hiển thị trong [Listing 4-2,](#page-52-1) chúng ta đã bao gồm các bit khác (O_CREAT, O_TRUNC, và O_APPEND) trong flags ngoài chế độ truy cập file. Bây giờ chúng ta xem xét đối số flags chi tiết hơn. Table 4-3 tóm tắt bộ hằng số đầy đủ có thể được ORed theo bit wise (|) trong flags. Cột cuối cùng cho biết những hằng số nào được chuẩn hóa trong SUSv3 hoặc SUSv4.

**Table 4-3:** Giá trị cho đối số flags của open()

| Flag        | Mục đích                                                           | SUS? |
|-------------|------------------------------------------------------------------|------|
| O_RDONLY    | Mở để chỉ đọc                                                    | v3   |
| O_WRONLY    | Mở để chỉ ghi                                                    | v3   |
| O_RDWR      | Mở để đọc và ghi                                                 | v3   |
| O_CLOEXEC   | Đặt cờ close-on-exec (kể từ Linux 2.6.23)                        | v4   |
| O_CREAT     | Tạo file nếu nó chưa tồn tại                                     | v3   |
| O_DIRECT    | File I/O bypass buffer cache                                     |      |
| O_DIRECTORY | Fail nếu pathname không phải là directory                        | v4   |
| O_EXCL      | Với O_CREAT: tạo file một cách độc quyền                         | v3   |
| O_LARGEFILE | Được sử dụng trên các hệ thống 32-bit để mở các file lớn         |      |
| O_NOATIME   | Không cập nhật thời gian truy cập file cuối cùng trên read() (kể từ Linux 2.6.8) |      |
| O_NOCTTY    | Không cho phép pathname trở thành controlling terminal           | v3   |
| O_NOFOLLOW  | Không dereference symbolic link                                  | v4   |
| O_TRUNC     | Cắt ngắn file hiện tại thành độ dài bằng không                   | v3   |
| O_APPEND    | Write luôn được nối thêm vào cuối file                           | v3   |
| O_ASYNC     | Tạo signal khi I/O có thể                                        |      |
| O_DSYNC     | Cung cấp synchronized I/O data integrity (kể từ Linux 2.6.33)    | v3   |
| O_NONBLOCK  | Mở ở chế độ nonblocking                                           | v3   |
| O_SYNC      | Làm cho file write đồng bộ                                        | v3   |

Các hằng số trong Table 4-3 được chia thành các nhóm sau:

-  File access mode flags: Đây là các flag O_RDONLY, O_WRONLY, và O_RDWR được mô tả trước đây. Chúng có thể được truy xuất bằng cách sử dụng hoạt động `fcntl()` F_GETFL [\(Section 5.3](#page-72-0)).
-  File creation flags: Đây là các flag được hiển thị trong phần thứ hai của Table 4-3. Chúng kiểm soát các khía cạnh khác nhau của hành vi của lệnh gọi `open()`, cũng như các tùy chọn cho các hoạt động I/O tiếp theo. Các flag này không thể được truy xuất hoặc thay đổi.
-  Open file status flags: Đây là các flag còn lại trong Table 4-3. Chúng có thể được truy xuất và sửa đổi bằng cách sử dụng các hoạt động `fcntl()` F_GETFL và F_SETFL ([Sec](#page-72-0)[tion 5.3](#page-72-0)). Các flag này đôi khi được gọi là file status flags.

Kể từ kernel 2.6.22, các file dành riêng cho Linux trong thư mục /proc/PID/fdinfo có thể được đọc để lấy thông tin về file descriptor của bất kỳ process nào trên hệ thống. Có một file trong thư mục này cho mỗi open file descriptor của process, với một tên phù hợp với số của descriptor. Trường pos trong file này hiển thị current file offset [\(Section 4.7](#page-60-1)). Trường flags là một số octal cho thấy file access mode flags và open file status flags. (Để giải mã số này, chúng ta cần xem các giá trị số của các flag này trong các tệp header thư viện C.)

Chi tiết cho các hằng số flags như sau:

#### O_APPEND

Write luôn được nối thêm vào cuối file. Chúng ta thảo luận về ý nghĩa của flag này trong [Section 5.1.](#page-69-0)

#### O_ASYNC

Tạo signal khi I/O trở thành có thể trên file descriptor được trả về bởi `open()`. Tính năng này, được gọi là signal-driven I/O, chỉ có sẵn cho các loại file nhất định, chẳng hạn như terminals, FIFOs, và sockets. (Flag O_ASYNC không được chỉ định trong SUSv3; tuy nhiên, nó, hoặc đồng nghĩa cũ hơn, FASYNC, được tìm thấy trên hầu hết các triển khai UNIX.) Trên Linux, chỉ định flag O_ASYNC khi gọi `open()` không có tác dụng. Để bật signal-driven I/O, chúng ta phải thay vào đó đặt flag này bằng cách sử dụng hoạt động `fcntl()` F_SETFL [\(Section 5.3](#page-72-0)). (Một số triển khai UNIX khác hoạt động tương tự.) Tham khảo Section 63.3 để biết thêm thông tin về flag O_ASYNC.

#### O_CLOEXEC (kể từ Linux 2.6.23)

Bật cờ close-on-exec (FD_CLOEXEC) cho file descriptor mới. Chúng ta mô tả cờ FD_CLOEXEC trong Section 27.4. Sử dụng flag O_CLOEXEC cho phép chương trình tránh các hoạt động `fcntl()` F_SETFD và F_SETFD bổ sung để đặt cờ close-on-exec. Nó cũng cần thiết trong các chương trình multithreaded để tránh các race condition có thể xảy ra khi sử dụng kỹ thuật sau. Các race này có thể xảy ra khi một thread mở file descriptor và sau đó cố gắng đánh dấu nó close-on-exec cùng lúc với thread khác thực hiện `fork()` và sau đó là `exec()` của một chương trình tùy ý. (Giả sử rằng thread thứ hai quản lý để cả `fork()` và `exec()` giữa thời gian thread đầu tiên mở file descriptor và sử dụng `fcntl()` để đặt cờ close-on-exec.) Các race như vậy có thể dẫn đến các file descriptor mở được vô tình chuyển đến các chương trình không an toàn. (Chúng ta nói thêm về race condition trong [Section 5.1](#page-69-0).)

#### O_CREAT

Nếu file chưa tồn tại, nó sẽ được tạo dưới dạng một file mới rỗng. Flag này có hiệu lực ngay cả khi file chỉ được mở để đọc. Nếu chúng ta chỉ định O_CREAT, thì chúng ta phải cung cấp đối số mode trong lệnh gọi `open()`; nếu không, quyền hạn của file mới sẽ được đặt thành một giá trị ngẫu nhiên từ stack.

#### O_DIRECT

Cho phép file I/O bypass buffer cache. Tính năng này được mô tả trong Section 13.6. Macro kiểm tra tính năng `_GNU_SOURCE` phải được định nghĩa để làm cho định nghĩa hằng số này khả dụng từ <fcntl.h>.

#### O_DIRECTORY

Trả về lỗi (`errno` bằng ENOTDIR) nếu pathname không phải là directory. Flag này là một phần mở rộng được thiết kế cụ thể để triển khai `opendir()` (Section 18.8). Macro kiểm tra tính năng `_GNU_SOURCE` phải được định nghĩa để làm cho định nghĩa flag này khả dụng từ <fcntl.h>.

#### O_DSYNC (kể từ Linux 2.6.33)

Thực hiện file write theo yêu cầu của synchronized I/O data integrity completion. Xem thảo luận về kernel I/O buffering trong Section 13.3.

#### O_EXCL

Flag này được sử dụng kết hợp với O_CREAT để cho biết rằng nếu file đã tồn tại, nó không nên được mở; thay vào đó, `open()` nên fail, với `errno` được đặt thành EEXIST. Nói cách khác, flag này cho phép người gọi đảm bảo rằng nó là process tạo file. Kiểm tra sự tồn tại và tạo file được thực hiện một cách atomic. Chúng ta thảo luận về khái niệm atomicity trong [Section 5.1](#page-69-0). Khi cả O_CREAT và O_EXCL được chỉ định trong flags, `open()` fail (với lỗi EEXIST) nếu pathname là một symbolic link. SUSv3 yêu cầu hành vi này để một ứng dụng có đặc quyền có thể tạo file ở một vị trí đã biết mà không có khả năng một symbolic link sẽ khiến file được tạo ở một vị trí khác (ví dụ: thư mục hệ thống), điều này sẽ có ý nghĩa bảo mật.

#### O_LARGEFILE

Mở file với hỗ trợ file lớn. Flag này được sử dụng trên các hệ thống 32-bit để làm việc với các file lớn. Mặc dù nó không được chỉ định trong SUSv3, flag O_LARGEFILE được tìm thấy trên một số triển khai UNIX khác. Trên các triển khai Linux 64 bit như Alpha và IA-64, flag này không có tác dụng. Xem [Section 5.10](#page-83-0) để biết thêm thông tin.

#### O_NOATIME (kể từ Linux 2.6.8)

Không cập nhật thời gian truy cập file cuối cùng (trường `st_atime` được mô tả trong Section 15.1) khi đọc từ file này. Để sử dụng flag này, effective user ID của process gọi phải khớp với chủ sở hữu file, hoặc process phải có đặc quyền (CAP_FOWNER); nếu không, `open()` fail với lỗi EPERM. (Trong thực tế, đối với một process không có đặc quyền, đó là file-system user ID của process, chứ không phải effective user ID của nó, phải khớp với user ID của file khi mở file với flag O_NOATIME, như được mô tả trong Section 9.5.) Flag này là một phần mở rộng Linux không chuẩn. Để công khai định nghĩa của nó từ <fcntl.h>, chúng ta phải định nghĩa macro kiểm tra tính năng `_GNU_SOURCE`. Flag O_NOATIME được dự định để sử dụng bởi các chương trình indexing và backup. Việc sử dụng nó có thể làm giảm đáng kể lượng hoạt động disk, bởi vì các lần tìm kiếm disk liên tục sang sau trên toàn disk không được yêu cầu để đọc nội dung file và cập nhật thời gian truy cập cuối cùng trong i-node của file (Section 14.4). Chức năng tương tự O_NOATIME có sẵn bằng cách sử dụng flag `MS_NOATIME` `mount()` (Section 14.8.1) và flag `FS_NOATIME_FL` (Section 15.5).

#### O_NOCTTY

Nếu file đang được mở là một terminal device, ngăn chặn nó trở thành controlling terminal. Các controlling terminal được thảo luận trong Section 34.4. Nếu file đang được mở không phải là terminal, flag này không có tác dụng.

#### O_NOFOLLOW

Thông thường, `open()` dereference pathname nếu nó là một symbolic link. Tuy nhiên, nếu flag O_NOFOLLOW được chỉ định, thì `open()` fail (với `errno` được đặt thành ELOOP) nếu pathname là một symbolic link. Flag này hữu ích, đặc biệt là trong các chương trình có đặc quyền, để đảm bảo rằng `open()` không dereference một symbolic link. Để công khai định nghĩa của flag này từ <fcntl.h>, chúng ta phải định nghĩa macro kiểm tra tính năng `_GNU_SOURCE`.

#### O_NONBLOCK

Mở file ở chế độ nonblocking. Xem [Section 5.9](#page-82-0).

#### O_SYNC

Mở file cho synchronous I/O. Xem thảo luận về kernel I/O buffering trong Section 13.3.

#### O_TRUNC

Nếu file đã tồn tại và là một regular file, thì cắt ngắn nó thành độ dài bằng không, hủy bất kỳ dữ liệu hiện tại nào. Trên Linux, cắt ngắn xảy ra cho dù file được mở để đọc hay ghi (trong cả hai trường hợp, chúng ta phải có quyền ghi trên file). SUSv3 để sự kết hợp của O_RDONLY và O_TRUNC chưa xác định, nhưng hầu hết các triển khai UNIX khác hoạt động giống như Linux.

## **4.3.2 Lỗi từ open()**

Nếu xảy ra lỗi khi cố gắng mở file, `open()` trả về –1, và `errno` xác định nguyên nhân của lỗi. Sau đây là một số lỗi có thể xảy ra (ngoài những lỗi đã được lưu ý khi mô tả đối số flags ở trên):

#### EACCES

Quyền hạn file không cho phép process gọi mở file ở chế độ được chỉ định bởi flags. Ngoài ra, vì quyền hạn directory, file không thể được truy cập, hoặc file không tồn tại và không thể được tạo.

EISDIR

File được chỉ định là một directory, và người gọi cố gắng mở nó để ghi. Điều này không được phép. (Mặt khác, có những dịp khi có thể hữu ích để mở directory để đọc. Chúng ta cân nhắc một ví dụ trong Section 18.11.)

EMFILE

Giới hạn resource của process trên số file descriptor mở đã được đạt tới (RLIMIT_NOFILE, được mô tả trong Section 36.3).

ENFILE

Giới hạn toàn hệ thống trên số file mở đã được đạt tới.

ENOENT

File được chỉ định không tồn tại, và O_CREAT không được chỉ định, hoặc O_CREAT được chỉ định, và một trong các directory trong pathname không tồn tại hoặc là symbolic link chỉ đến pathname không tồn tại (một link dangling).

EROFS

File được chỉ định nằm trên read-only file system và người gọi cố gắng mở nó để ghi.

ETXTBSY

File được chỉ định là một executable file (một chương trình) hiện đang được thực thi. Nó không được phép sửa đổi (tức là, mở để ghi) file executable liên quan đến một chương trình đang chạy. (Chúng ta phải kết thúc chương trình trước để có thể sửa đổi file executable.)

Khi chúng ta sau này mô tả các system call khác hoặc hàm library, chúng ta thường sẽ không liệt kê phạm vi các lỗi có thể xảy ra theo cách trên. (Danh sách như vậy có thể được tìm thấy trên trang manual tương ứng cho mỗi system call hoặc hàm library.) Chúng ta làm như vậy ở đây vì hai lý do. Một trong những lý do này là `open()` là system call đầu tiên được chúng ta mô tả chi tiết, và danh sách trên minh họa rằng một system call hoặc hàm library có thể fail vì bất kỳ lý do nào. Thứ hai, những lý do cụ thể tại sao `open()` có thể fail tạo thành một danh sách thú vị trong chính nó, minh họa một số yếu tố và kiểm tra đi vào khi một file được truy cập. (Danh sách trên không đầy đủ: xem trang manual open(2) để biết thêm lý do tại sao `open()` có thể fail.)

# **4.3.3 System Call creat()**

Trong các triển khai UNIX sớm, `open()` chỉ có hai đối số và không thể được sử dụng để tạo một file mới. Thay vào đó, system call `creat()` được sử dụng để tạo và mở một file mới.

```
#include <fcntl.h>
int creat(const char *pathname, mode_t mode);
                                          Returns file descriptor, or –1 on error
```

System call `creat()` tạo và mở một file mới với pathname đã cho, hoặc nếu file đã tồn tại, mở file và cắt ngắn nó thành độ dài bằng không. Như kết quả của hàm, `creat()` trả về một file descriptor có thể được sử dụng trong các system call tiếp theo. Gọi `creat()` tương đương với lệnh gọi `open()` sau:

```
fd = open(pathname, O_WRONLY | O_CREAT | O_TRUNC, mode);
```

Vì đối số flags của `open()` cung cấp kiểm soát lớn hơn trên cách file được mở (ví dụ: chúng ta có thể chỉ định O_RDWR thay vì O_WRONLY), `creat()` bây giờ đã lỗi thời, mặc dù nó vẫn có thể được nhìn thấy trong các chương trình cũ.

# **4.4 Đọc từ File: read()**

System call `read()` đọc dữ liệu từ file mở được tham chiếu bởi descriptor fd.

```
#include <unistd.h>
ssize_t read(int fd, void *buffer, size_t count);
                        Returns number of bytes read, 0 on EOF, or –1 on error
```

Đối số count chỉ định số byte tối đa để đọc. (Kiểu dữ liệu `size_t` là một kiểu số nguyên không có dấu.) Đối số buffer cung cấp địa chỉ của buffer bộ nhớ mà vào đó dữ liệu đầu vào sẽ được đặt. Buffer này phải dài ít nhất count byte.

> System call không cấp phát bộ nhớ cho các buffer được sử dụng để trả về thông tin cho người gọi. Thay vào đó, chúng ta phải chuyển một con trỏ đến một buffer bộ nhớ được cấp phát trước đó có kích thước chính xác. Điều này trái ngược với một số hàm library có cấp phát buffer bộ nhớ để trả về thông tin cho người gọi.

Lệnh gọi thành công `read()` trả về số byte thực tế được đọc, hoặc 0 nếu gặp end-of-file. Khi xảy ra lỗi, thường là –1 được trả về. Kiểu dữ liệu `ssize_t` là một kiểu số nguyên có dấu được sử dụng để lưu giữ byte count hoặc chỉ báo lỗi –1.

Lệnh gọi `read()` có thể đọc ít hơn số byte được yêu cầu. Đối với regular file, lý do có thể xảy ra là chúng ta gần đến cuối file.

Khi `read()` được áp dụng cho các loại file khác—chẳng hạn như pipes, FIFOs, sockets, hoặc terminals—cũng có nhiều trường hợp khác nhau mà nó có thể đọc ít hơn byte được yêu cầu. Ví dụ, theo mặc định, `read()` từ terminal đọc ký tự chỉ tới ký tự newline (\n) tiếp theo. Chúng ta xem xét các trường hợp này khi chúng ta xử lý các loại file khác trong các chương tiếp theo.

Sử dụng `read()` để đầu vào một loạt ký tự từ, chẳng hạn như, một terminal, chúng ta có thể mong đợi mã sau đây hoạt động:

```
#define MAX_READ 20
char buffer[MAX_READ];
if (read(STDIN_FILENO, buffer, MAX_READ) == -1)
 errExit("read");
printf("The input data was: %s\n", buffer);
```

Đầu ra từ đoạn mã này có khả năng sẽ kỳ lạ, vì nó có thể sẽ bao gồm các ký tự ngoài chuỗi thực tế được nhập. Điều này là vì `read()` không đặt byte null kết thúc ở cuối chuỗi được `printf()` yêu cầu in. Một suy nghĩ kỹ lưỡng dẫn chúng ta nhận ra rằng điều này phải là như vậy, vì `read()` có thể được sử dụng để đọc bất kỳ chuỗi byte nào từ file. Trong một số trường hợp, đầu vào này có thể là text, nhưng trong các trường hợp khác, đầu vào có thể là số nguyên nhị phân hoặc C structure ở dạng nhị phân. Không có cách nào cho `read()` để phân biệt, và vì vậy nó không thể tuân theo quy ước C của chuỗi ký tự kết thúc null. Nếu byte null kết thúc được yêu cầu ở cuối buffer đầu vào, chúng ta phải đặt nó ở đó một cách rõ ràng:

```
char buffer[MAX_READ + 1];
ssize_t numRead;
numRead = read(STDIN_FILENO, buffer, MAX_READ);
if (numRead == -1)
 errExit("read");
buffer[numRead] = '\0';
printf("The input data was: %s\n", buffer);
```

Vì byte null kết thúc yêu cầu một byte bộ nhớ, kích thước của buffer phải lớn hơn ít nhất một byte so với chuỗi lớn nhất chúng ta mong đợi đọc.

# **4.5 Ghi vào File: write()**

System call `write()` ghi dữ liệu vào file mở.

```
#include <unistd.h>
ssize_t write(int fd, void *buffer, size_t count);
                                Returns number of bytes written, or –1 on error
```

Các đối số cho `write()` tương tự như đối với `read()`: buffer là địa chỉ của dữ liệu được ghi; count là số byte được ghi từ buffer; và fd là file descriptor tham chiếu đến file mà dữ liệu được ghi vào.

Khi thành công, `write()` trả về số byte thực tế được ghi; điều này có thể ít hơn count. Đối với disk file, các lý do có thể xảy ra cho một partial write là disk đã được điền đầy hoặc giới hạn resource của process trên kích thước file đã được đạt tới. (Giới hạn liên quan là RLIMIT_FSIZE, được mô tả trong Section 36.3.)

Khi thực hiện I/O trên disk file, một return thành công từ `write()` không đảm bảo rằng dữ liệu đã được chuyển sang disk, vì kernel thực hiện buffering của disk I/O để giảm hoạt động disk và đẩy nhanh các lệnh gọi `write()`. Chúng ta xem xét các chi tiết trong Chapter 13.

# **4.6 Đóng File: close()**

System call `close()` đóng một file descriptor mở, giải phóng nó để sử dụng lại tiếp theo bởi process. Khi process kết thúc, tất cả các file descriptor mở của nó sẽ tự động đóng.

```
#include <unistd.h>
int close(int fd);
                                              Returns 0 on success, or –1 on error
```

Nó thường là một thực hành tốt để đóng rõ ràng các file descriptor không cần thiết, vì điều này làm cho mã của chúng ta dễ đọc và đáng tin cậy khi gặp các sửa đổi sau. Hơn nữa, file descriptor là một resource tiêu thụ, vì vậy không đóng file descriptor có thể dẫn đến process hết descriptor. Đây là một vấn đề đặc biệt quan trọng khi viết các chương trình dài hạn xử lý nhiều file, chẳng hạn như shell hoặc network server.

Giống như mọi system call khác, lệnh gọi `close()` phải được bọc trong mã kiểm tra lỗi, chẳng hạn như sau:

```
if (close(fd) == -1)
 errExit("close");
```

Điều này bắt các lỗi như cố gắng đóng một file descriptor không mở hoặc đóng cùng một file descriptor hai lần, và bắt các điều kiện lỗi mà một file system cụ thể có thể chẩn đoán trong hoạt động close.

> NFS (Network File System) cung cấp một ví dụ về lỗi cụ thể cho file system. Nếu xảy ra lỗi commit NFS, có nghĩa là dữ liệu không đạt được disk từ xa, thì lỗi này được truyền tới ứng dụng dưới dạng lỗi trong lệnh gọi `close()`.

## <span id="page-60-1"></span>**4.7 Thay đổi File Offset: lseek()**

<span id="page-60-0"></span>Đối với mỗi file mở, kernel ghi lại file offset, đôi khi cũng được gọi là read-write offset hoặc pointer. Đây là vị trí trong file mà tại đó `read()` hoặc `write()` tiếp theo sẽ bắt đầu. File offset được biểu thị dưới dạng vị trí byte thứ tự tương đối so với đầu file. Byte đầu tiên của file nằm ở offset 0.

File offset được đặt để trỏ đến đầu file khi file được mở và tự động được điều chỉnh bởi mỗi lệnh gọi `read()` hoặc `write()` tiếp theo để nó trỏ đến byte tiếp theo của file sau byte(s) vừa được đọc hoặc ghi. Do đó, các lệnh gọi `read()` và `write()` liên tiếp tiến bộ tuần tự thông qua file.

System call `lseek()` điều chỉnh file offset của file mở được tham chiếu bởi file descriptor fd, theo các giá trị được chỉ định trong offset và whence.

```
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
                               Returns new file offset if successful, or –1 on error
```

Đối số offset chỉ định một giá trị tính bằng byte. (Kiểu dữ liệu `off_t` là kiểu số nguyên có dấu được chỉ định bởi SUSv3.) Đối số whence cho biết điểm cơ sở từ đó offset sẽ được diễn giải, và là một trong các giá trị sau:

SEEK_SET

File offset được đặt offset byte từ đầu file.

SEEK_CUR

File offset được điều chỉnh bởi offset byte so với file offset hiện tại.

SEEK_END

File offset được đặt thành kích thước file cộng với offset. Nói cách khác, offset được diễn giải với respect tới byte tiếp theo sau byte cuối cùng của file.

Figure 4-1 cho thấy cách diễn giải đối số whence.

Trong các triển khai UNIX trước, các số nguyên 0, 1, và 2 được sử dụng, chứ không phải các hằng số SEEK_\* được hiển thị trong văn bản chính. Các phiên bản BSD cũ sử dụng các tên khác nhau cho các giá trị này: L_SET, L_INCR, và L_XTND.

```txt
                      File containing N bytes of data
        <-------------------------------------------------------->

byte
number      0     1     ...     ...     N-2     N-1     N     N+1
           +-----+-----+-----+-----+-----+-----+-----+-----+-----+
           |  0  |  1  | ... | ... | N-2 | N-1 |     |     |     |
           +-----+-----+-----+-----+-----+-----+-----+-----+-----+
                                                  ^     ^
                                                  |     |
                                           Unwritten bytes
                                               past EOF

                              ^
                              |
                       Current file offset
```

**Figure 4-1:** Diễn giải đối số whence của lseek()

Nếu whence là SEEK_CUR hoặc SEEK_END, offset có thể là âm hoặc dương; đối với SEEK_SET, offset phải không âm.

Giá trị return từ một lseek() thành công là file offset mới. Lệnh gọi sau đây truy xuất vị trí hiện tại của file offset mà không thay đổi nó:

```
curr = lseek(fd, 0, SEEK_CUR);
```

Một số triển khai UNIX (nhưng không phải Linux) có hàm không chuẩn `tell(fd)`, phục vụ cùng mục đích như lệnh gọi `lseek()` ở trên.

Dưới đây là một số ví dụ khác về các lệnh gọi `lseek()`, cùng với các bình luận cho biết vị trí file offset được di chuyển đến:

```
lseek(fd, 0, SEEK_SET); /* Start of file */
lseek(fd, 0, SEEK_END); /* Next byte after the end of the file */
lseek(fd, -1, SEEK_END); /* Last byte of file */
lseek(fd, -10, SEEK_CUR); /* Ten bytes prior to current location */
lseek(fd, 10000, SEEK_END); /* 10001 bytes past last byte of file */
```

Gọi `lseek()` chỉ đơn giản điều chỉnh bản ghi kernel của file offset liên quan đến file descriptor. Nó không gây ra bất kỳ truy cập thiết bị vật lý nào.

Chúng ta mô tả một số chi tiết bổ sung của mối quan hệ giữa file offset, file descriptor, và file mở trong Section [5.4](#page-73-0).

Chúng ta không thể áp dụng `lseek()` cho tất cả các loại file. Áp dụng `lseek()` cho pipe, FIFO, socket, hoặc terminal không được phép; `lseek()` fail, với `errno` được đặt thành ESPIPE. Mặt khác, có thể áp dụng `lseek()` cho các device mà nơi có ý nghĩa. Ví dụ, có thể tìm kiếm đến vị trí được chỉ định trên device disk hoặc tape.

> Chữ "l" trong tên `lseek()` bắt nguồn từ thực tế là đối số offset và giá trị return ban đầu được gõ dưới dạng long. Các triển khai UNIX sớm cung cấp system call `seek()`, gõ các giá trị này dưới dạng int.

#### **File hole**

Điều gì sẽ xảy ra nếu một chương trình tìm kiếm quá cuối file, và sau đó thực hiện I/O? Lệnh gọi `read()` sẽ trả về 0, cho biết end-of-file. Điều gì đó đáng ngạc nhiên là có thể ghi byte ở một điểm tùy ý quá cuối file.

Không gian giữa end của file trước đó và byte được ghi mới được gọi là file hole. Từ quan điểm lập trình, các byte trong hole tồn tại, và đọc từ hole trả về một buffer của byte chứa 0 (byte null).

Tuy nhiên, file hole không chiếm bất kỳ không gian disk nào. File system không cấp phát bất kỳ disk block nào cho hole cho đến khi, tại một số điểm sau, dữ liệu được ghi vào nó. Ưu điểm chính của file hole là một file thưa thớt chiếm ít không gian disk hơn so với nếu các byte null thực sự cần được cấp phát trong các disk block. File Core dump (Section 22.1) là những ví dụ phổ biến về file chứa các hole lớn.

> Tuyên bố rằng file hole không tiêu thụ không gian disk cần được đủ tiêu chuẩn. Trên hầu hết các file system, không gian file được cấp phát theo đơn vị block (Section 14.3). Kích thước của block tùy thuộc vào file system, nhưng thường là khoảng 1024, 2048, hoặc 4096 byte. Nếu cạnh của hole rơi vào giữa block, chứ không phải trên ranh giới block, thì một block hoàn chỉnh được cấp phát để lưu trữ dữ liệu trong phần khác của block, và phần tương ứng với hole được điền bằng byte null.

Hầu hết các native UNIX file system hỗ trợ khái niệm file hole, nhưng nhiều nonnative file system (ví dụ: VFAT của Microsoft) không. Trên file system không hỗ trợ hole, các byte null rõ ràng được ghi vào file.

Sự tồn tại của hole có nghĩa là kích thước danh nghĩa của file có thể lớn hơn lượng lưu trữ disk mà nó sử dụng (trong một số trường hợp, lớn hơn đáng kể). Ghi byte vào giữa hole của file sẽ giảm lượng không gian disk miễn phí khi kernel cấp phát các block để điền hole, mặc dù kích thước file không thay đổi. Kịch bản như vậy không phổ biến, nhưng vẫn là cái cần phải biết.

> SUSv3 chỉ định một hàm, `posix_fallocate(fd, offset, len)`, đảm bảo rằng không gian được cấp phát trên disk cho phạm vi byte được chỉ định bởi offset và len cho file disk được tham chiếu bởi descriptor fd. Điều này cho phép một ứng dụng chắc chắn rằng một `write()` sau này vào file sẽ không fail vì không gian disk bị cạn kiệt (điều này có thể xảy ra nếu một hole trong file được điền, hoặc một số ứng dụng khác tiêu thụ không gian trên disk). Về mặt lịch sử, việc triển khai glibc của hàm này đạt được kết quả mong muốn bằng cách ghi một byte 0 vào mỗi block trong phạm vi được chỉ định. Kể từ phiên bản 2.6.23, Linux cung cấp system call `fallocate()`, cung cấp một cách hiệu quả hơn để đảm bảo rằng không gian cần thiết được cấp phát, và việc triển khai glibc `posix_fallocate()` sử dụng system call này khi nó khả dụng.

Section 14.4 mô tả cách các hole được biểu diễn trong file, và Section 15.1 mô tả system call `stat()`, có thể cho chúng ta biết kích thước hiện tại của file, cũng như số block thực tế được cấp phát cho file.

#### **Chương trình ví dụ**

[Listing 4-3](#page-63-0) minh họa cách sử dụng `lseek()` kết hợp với `read()` và `write()`. Đối số dòng lệnh đầu tiên cho chương trình này là tên của file được mở. Các đối số còn lại chỉ định các hoạt động I/O được thực hiện trên file. Mỗi hoạt động này bao gồm một chữ cái theo sau là một giá trị liên quan (không có khoảng trắng ngăn cách):

-  `soffset`: Tìm kiếm tới byte offset từ đầu file.
-  `rlength`: Đọc length byte từ file, bắt đầu từ current file offset, và hiển thị chúng ở dạng text.
-  `Rlength`: Đọc length byte từ file, bắt đầu từ current file offset, và hiển thị chúng ở dạng hexadecimal.
-  `wstr`: Ghi chuỗi ký tự được chỉ định trong str tại current file offset.

<span id="page-63-0"></span>**Listing 4-3:** Minh họa read(), write(), và lseek()

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– fileio/seek_io.c
#include <sys/stat.h>
#include <fcntl.h>
#include <ctype.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 size_t len;
 off_t offset;
 int fd, ap, j;
 char *buf;
 ssize_t numRead, numWritten;
 if (argc < 3 || strcmp(argv[1], "--help") == 0)
 usageErr("%s file {r<length>|R<length>|w<string>|s<offset>}...\n",
 argv[0]);
 fd = open(argv[1], O_RDWR | O_CREAT,
 S_IRUSR | S_IWUSR | S_IRGRP | S_IWGRP |
 S_IROTH | S_IWOTH); /* rw-rw-rw- */
 if (fd == -1)
 errExit("open");
 for (ap = 2; ap < argc; ap++) {
 switch (argv[ap][0]) {
 case 'r': /* Display bytes at current offset, as text */
 case 'R': /* Display bytes at current offset, in hex */
 len = getLong(&argv[ap][1], GN_ANY_BASE, argv[ap]);
```

```
 buf = malloc(len);
 if (buf == NULL)
 errExit("malloc");
 numRead = read(fd, buf, len);
 if (numRead == -1)
 errExit("read");
 if (numRead == 0) {
 printf("%s: end-of-file\n", argv[ap]);
 } else {
 printf("%s: ", argv[ap]);
 for (j = 0; j < numRead; j++) {
 if (argv[ap][0] == 'r')
 printf("%c", isprint((unsigned char) buf[j]) ?
 buf[j] : '?');
 else
 printf("%02x ", (unsigned int) buf[j]);
 }
 printf("\n");
 }
 free(buf);
 break;
 case 'w': /* Write string at current offset */
 numWritten = write(fd, &argv[ap][1], strlen(&argv[ap][1]));
 if (numWritten == -1)
 errExit("write");
 printf("%s: wrote %ld bytes\n", argv[ap], (long) numWritten);
 break;
 case 's': /* Change file offset */
 offset = getLong(&argv[ap][1], GN_ANY_BASE, argv[ap]);
 if (lseek(fd, offset, SEEK_SET) == -1)
 errExit("lseek");
 printf("%s: seek succeeded\n", argv[ap]);
 break;
 default:
 cmdLineErr("Argument must start with [rRws]: %s\n", argv[ap]);
 }
 }
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– fileio/seek_io.c
```

Phiên bản log shell session sau đây minh họa cách sử dụng chương trình trong Listing [4-3,](#page-63-0) cho thấy điều gì xảy ra khi chúng ta cố gắng đọc byte từ file hole:

```
$ touch tfile Create new, empty file
$ ./seek_io tfile s100000 wabc Seek to offset 100,000, write "abc"
s100000: seek succeeded
wabc: wrote 3 bytes
```

\$ **ls -l tfile** Check size of file -rw-r--r-- 1 mtk users 100003 Feb 10 10:35 tfile s10000: seek succeeded

\$ **./seek_io tfile s10000 R5** Seek to offset 10,000, read 5 bytes from hole

R5: 00 00 00 00 00 Bytes in the hole contain 0

# <span id="page-65-0"></span>**4.8 Các hoạt động ngoài mô hình I/O phổ quát: ioctl()**

System call `ioctl()` là một cơ chế thông qua để thực hiện các hoạt động file và device ngoài mô hình I/O phổ quát được mô tả trước đó trong chương này.

```
#include <sys/ioctl.h>
int ioctl(int fd, int request, ... /* argp */);
                   Value returned on success depends on request, or –1 on error
```

Đối số fd là một file descriptor mở cho device hoặc file mà trên đó hoạt động kiểm soát được chỉ định bởi request sẽ được thực hiện. Các tệp header cụ thể của device định nghĩa các hằng số có thể được chuyển trong đối số request.

Như được chỉ bởi ký hiệu dấu chấm lửng tiêu chuẩn C (...), đối số thứ ba cho `ioctl()`, chúng ta gắn nhãn argp, có thể là bất kỳ loại nào. Giá trị của đối số request cho phép `ioctl()` xác định loại giá trị nào để mong đợi trong argp. Thông thường, argp là con trỏ đến một số nguyên hoặc cấu trúc; trong một số trường hợp, nó không được sử dụng.

Chúng ta sẽ thấy một số cách sử dụng cho `ioctl()` trong các chương sau (xem, ví dụ, Section 15.5).

> Đặc tả duy nhất mà SUSv3 đưa ra cho `ioctl()` là các hoạt động kiểm soát các device STREAM. (Cơ sở STREAM là một tính năng System V không được hỗ trợ bởi kernel Linux mainline, mặc dù một vài triển khai add-on đã được phát triển.) Không có hoạt động `ioctl()` nào khác được mô tả trong cuốn sách này được chỉ định trong SUSv3. Tuy nhiên, lệnh gọi `ioctl()` đã là một phần của hệ thống UNIX kể từ các phiên bản sớm, và do đó một số hoạt động `ioctl()` mà chúng ta mô tả được cung cấp trên nhiều triển khai UNIX khác. Khi chúng ta mô tả từng hoạt động `ioctl()`, chúng ta lưu ý các vấn đề về tính di động.

# **4.9 Tóm tắt**

Để thực hiện I/O trên regular file, chúng ta phải trước tiên lấy file descriptor bằng cách sử dụng `open()`. I/O sau đó được thực hiện bằng cách sử dụng `read()` và `write()`. Sau khi thực hiện tất cả I/O, chúng ta sẽ giải phóng file descriptor và các resource liên quan của nó bằng cách sử dụng `close()`. Các system call này có thể được sử dụng để thực hiện I/O trên tất cả các loại file.

Thực tế là tất cả các loại file và device driver triển khai cùng một giao diện I/O cho phép tính phổ quát của I/O, có nghĩa là một chương trình thường có thể được sử dụng với bất kỳ loại file nào mà không yêu cầu mã cụ thể cho loại file.

Đối với mỗi file mở, kernel duy trì một file offset, xác định vị trí mà tại đó `read()` hoặc `write()` tiếp theo sẽ xảy ra. File offset được ngầm cập nhật bởi các phép đọc và ghi. Sử dụng `lseek()`, chúng ta có thể rõ ràng định vị lại file offset sang bất kỳ vị trí nào trong file hoặc quá cuối file. Ghi dữ liệu tại vị trí vượt quá end trước đó của file tạo ra một hole trong file. Đọc từ file hole trả về các byte chứa zero.

System call `ioctl()` là một catchall cho các hoạt động device và file không phù hợp với mô hình I/O file tiêu chuẩn.

## **4.10 Bài tập**

- **4-1.** Lệnh tee đọc standard input của nó cho đến end-of-file, ghi một bản sao của đầu vào vào standard output và vào file được đặt tên trong đối số dòng lệnh của nó. (Chúng ta hiển thị một ví dụ về cách sử dụng lệnh này khi chúng ta thảo luận về FIFOs trong Section 44.7.) Triển khai tee bằng cách sử dụng I/O system call. Theo mặc định, tee ghi đè lên bất kỳ file hiện tại nào với tên nhất định. Triển khai tùy chọn dòng lệnh –a (tee –a file), làm cho tee nối text vào cuối file nếu nó đã tồn tại. (Tham khảo Appendix B để mô tả hàm `getopt()`, có thể được sử dụng để phân tích các tùy chọn dòng lệnh.)
- **4-2.** Viết một chương trình giống như cp mà khi được sử dụng để sao chép một regular file chứa hole (chuỗi byte null), cũng tạo các hole tương ứng trong file đích.
