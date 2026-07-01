## Chương 4
# **FILE I/O: MÔ HÌNH I/O PHỔ QUÁT**

Chúng ta bắt đầu tìm hiểu nghiêm túc về system call API. File là điểm khởi đầu tốt, vì chúng đóng vai trò trung tâm trong triết lý UNIX. Trọng tâm của chương này là các system call dùng để thực hiện input và output trên file.

Chúng ta sẽ giới thiệu khái niệm file descriptor, sau đó xem xét các system call tạo nên cái gọi là mô hình I/O phổ quát. Đây là những system call dùng để mở và đóng file, cũng như đọc và ghi dữ liệu.

Chúng ta tập trung vào I/O trên disk file. Tuy nhiên, phần lớn nội dung trình bày ở đây còn liên quan đến các chương sau, vì cùng những system call đó được sử dụng để thực hiện I/O trên tất cả các loại file, chẳng hạn như pipe và terminal.

Chương [5](#page-68-0) mở rộng thảo luận trong chương này với thêm chi tiết về file I/O. Một khía cạnh khác của file I/O — buffering — đủ phức tạp để có chương riêng. Chương 13 đề cập đến I/O buffering trong kernel và trong thư viện stdio.

## **4.1 Tổng quan**

Tất cả các system call thực hiện I/O đều tham chiếu đến các file đang mở thông qua một file descriptor — một số nguyên không âm (thường có giá trị nhỏ). File descriptor được dùng để tham chiếu đến tất cả các loại file đang mở, bao gồm pipe, FIFO, socket, terminal, device và regular file. Mỗi process có tập file descriptor riêng của mình.

Theo quy ước, hầu hết các chương trình đều kỳ vọng có thể sử dụng ba standard file descriptor được liệt kê trong [Bảng 4-1](#page-49-0). Ba descriptor này được shell mở thay cho chương trình, trước khi chương trình khởi động. Hay nói chính xác hơn, chương trình kế thừa các bản sao file descriptor của shell, và shell thông thường luôn hoạt động với ba file descriptor này ở trạng thái mở. (Trong một interactive shell, ba file descriptor này thường tham chiếu đến terminal mà shell đang chạy trên đó.) Nếu các I/O redirection được chỉ định trên command line, shell sẽ đảm bảo rằng các file descriptor được sửa đổi phù hợp trước khi khởi động chương trình.

<span id="page-49-0"></span>**Bảng 4-1:** Standard file descriptor

| File descriptor | Mục đích        | Tên POSIX     | Luồng stdio  |
|-----------------|-----------------|---------------|--------------|
| 0               | standard input  | STDIN_FILENO  | stdin        |
| 1               | standard output | STDOUT_FILENO | stdout       |
| 2               | standard error  | STDERR_FILENO | stderr       |

Khi tham chiếu đến các file descriptor này trong chương trình, ta có thể dùng số (0, 1, hoặc 2) hoặc tốt hơn là dùng các tên chuẩn POSIX được định nghĩa trong `<unistd.h>`.

> Mặc dù các biến stdin, stdout và stderr ban đầu tham chiếu đến standard input, output và error của process, chúng có thể được thay đổi để trỏ đến bất kỳ file nào bằng cách sử dụng hàm thư viện freopen(). Trong quá trình hoạt động, freopen() có thể thay đổi file descriptor nằm bên dưới stream được mở lại. Nói cách khác, sau khi gọi freopen() trên stdout chẳng hạn, sẽ không còn an toàn khi giả định rằng file descriptor bên dưới vẫn là 1.

Bốn system call chính sau đây dùng để thực hiện file I/O (các ngôn ngữ lập trình và gói phần mềm thường chỉ gọi những system call này một cách gián tiếp, thông qua các thư viện I/O):

- `fd = open(pathname, flags, mode)` mở file được xác định bởi pathname, trả về một file descriptor dùng để tham chiếu đến file đang mở trong các lần gọi tiếp theo. Nếu file không tồn tại, open() có thể tạo file đó tùy theo cài đặt của tham số bitmask flags. Tham số flags cũng chỉ định file được mở để đọc, ghi, hay cả hai. Tham số mode chỉ định các permission được đặt lên file nếu file được tạo bởi lần gọi này. Nếu open() không được dùng để tạo file, tham số này bị bỏ qua và có thể được lược bỏ.
- `numread = read(fd, buffer, count)` đọc tối đa count byte từ file đang mở được tham chiếu bởi fd và lưu vào buffer. Lời gọi read() trả về số byte thực sự đã đọc. Nếu không còn byte nào để đọc (tức là đã gặp end-of-file), read() trả về 0.
- `numwritten = write(fd, buffer, count)` ghi tối đa count byte từ buffer vào file đang mở được tham chiếu bởi fd. Lời gọi write() trả về số byte thực sự đã ghi, có thể nhỏ hơn count.
- `status = close(fd)` được gọi sau khi tất cả I/O đã hoàn thành, nhằm giải phóng file descriptor fd và các tài nguyên kernel liên quan.

Trước khi đi vào chi tiết của các system call này, chúng ta cung cấp một minh họa ngắn về cách sử dụng chúng trong [Listing 4-1.](#page-50-0) Chương trình này là một phiên bản đơn giản của lệnh cp(1). Nó sao chép nội dung của file hiện có được đặt tên trong đối số command-line đầu tiên sang file mới được đặt tên trong đối số command-line thứ hai.

Ta có thể dùng chương trình trong [Listing 4-1](#page-50-0) như sau:

#### \$ **./copy oldfile newfile**

#### <span id="page-50-0"></span>**Listing 4-1:** Sử dụng các I/O system call

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

Một trong những đặc trưng nổi bật của mô hình UNIX I/O là khái niệm tính phổ quát của I/O. Điều này có nghĩa là cùng bốn system call — open(), read(), write() và close() — được dùng để thực hiện I/O trên tất cả các loại file, kể cả các device như terminal. Do đó, nếu ta viết một chương trình chỉ sử dụng những system call này, chương trình đó sẽ hoạt động được với bất kỳ loại file nào. Ví dụ, sau đây là các cách sử dụng hợp lệ của chương trình trong [Listing 4-1](#page-50-0):

```
$ ./copy test test.old          Sao chép một regular file
$ ./copy a.txt /dev/tty         Sao chép một regular file đến terminal này
$ ./copy /dev/tty b.txt         Sao chép input từ terminal này vào một regular file
$ ./copy /dev/pts/16 /dev/tty   Sao chép input từ một terminal khác
```

Tính phổ quát của I/O đạt được bằng cách đảm bảo rằng mỗi file system và device driver đều triển khai cùng một tập I/O system call. Vì các chi tiết đặc thù của file system hoặc device được xử lý bên trong kernel, chúng ta thường có thể bỏ qua các yếu tố đặc thù của device khi viết application. Khi cần truy cập các tính năng đặc thù của một file system hay device, chương trình có thể sử dụng system call tổng hợp ioctl() ([Mục 4.8](#page-65-0)), cung cấp giao diện cho các tính năng nằm ngoài mô hình I/O phổ quát.

# **4.3 Mở File: open()**

System call open() mở một file đã tồn tại hoặc tạo và mở một file mới.

```
#include <sys/stat.h>
#include <fcntl.h>
int open(const char *pathname, int flags, ... /* mode_t mode */);
                               Returns file descriptor on success, or –1 on error
```

File cần mở được xác định bởi tham số pathname. Nếu pathname là một symbolic link, nó sẽ được dereferenced. Khi thành công, open() trả về một file descriptor dùng để tham chiếu đến file trong các system call tiếp theo. Nếu xảy ra lỗi, open() trả về –1 và errno được đặt tương ứng.

Tham số flags là một bit mask chỉ định access mode cho file, sử dụng một trong các hằng số hiển thị trong [Bảng 4-2.](#page-52-0)

> Các phiên bản UNIX đời đầu sử dụng số 0, 1 và 2 thay vì các tên hiển thị trong Bảng [4-2](#page-52-0). Hầu hết các phiên bản UNIX hiện đại định nghĩa các hằng số này có giá trị đó. Vì vậy, ta thấy rằng O\_RDWR không tương đương với O\_RDONLY | O\_WRONLY; tổ hợp sau là một lỗi logic.

Khi open() được dùng để tạo file mới, tham số bit-mask mode chỉ định các permission được đặt lên file. (Kiểu dữ liệu mode\_t dùng để định kiểu mode là một integer type được quy định trong SUSv3.) Nếu lời gọi open() không chỉ định O\_CREAT, mode có thể được lược bỏ.

<span id="page-52-0"></span>**Bảng 4-2:** Các access mode của file

| Access mode | Mô tả                                           |
|-------------|-------------------------------------------------|
| O_RDONLY    | Mở file chỉ để đọc                              |
| O_WRONLY    | Mở file chỉ để ghi                              |
| O_RDWR      | Mở file để vừa đọc vừa ghi                      |

Ta mô tả file permission chi tiết trong Mục 15.4. Sau này ta sẽ thấy rằng permission thực sự được đặt trên file mới không chỉ phụ thuộc vào tham số mode, mà còn vào process umask (Mục 15.4.6) và default access control list (Mục 17.6) của thư mục cha (nếu có). Trong thời gian này, ta chỉ cần lưu ý rằng tham số mode có thể được chỉ định dưới dạng số (thường là octal) hoặc tốt hơn là bằng cách OR (|) các hằng số bit-mask được liệt kê trong Bảng 15-4, trang 295.

[Listing 4-2](#page-52-1) cho thấy các ví dụ sử dụng open(), một số trong đó dùng thêm các flag bit mà chúng ta mô tả ngay sau đây.

<span id="page-52-1"></span>**Listing 4-2:** Các ví dụ sử dụng open()

```
 /* Mở file hiện có để đọc */
 fd = open("startup", O_RDONLY);
 if (fd == -1)
 errExit("open");
   /* Mở file mới hoặc hiện có để đọc và ghi, truncate về zero
      byte; file permission đọc+ghi cho owner, không có gì cho những người khác */
 fd = open("myfile", O_RDWR | O_CREAT | O_TRUNC, S_IRUSR | S_IWUSR);
 if (fd == -1)
 errExit("open");
 /* Mở file mới hoặc hiện có để ghi; các lần ghi luôn
 append vào cuối file */
 fd = open("w.log", O_WRONLY | O_CREAT | O_TRUNC | O_APPEND,
 S_IRUSR | S_IWUSR);
 if (fd == -1)
 errExit("open");
```

#### **Số file descriptor được trả về bởi open()**

SUSv3 quy định rằng nếu open() thành công, nó đảm bảo sẽ sử dụng file descriptor có số thấp nhất chưa được dùng của process. Ta có thể tận dụng tính năng này để đảm bảo rằng một file được mở sử dụng một file descriptor cụ thể. Ví dụ, đoạn code sau đảm bảo file được mở bằng standard input (file descriptor 0).

```
if (close(STDIN_FILENO) == -1) /* Đóng file descriptor 0 */
 errExit("close");
fd = open(pathname, O_RDONLY);
if (fd == -1)
 errExit("open");
```

Vì file descriptor 0 không được sử dụng, open() được đảm bảo sẽ mở file bằng descriptor đó. Trong [Mục 5.5](#page-75-0), ta xem xét việc sử dụng dup2() và fcntl() để đạt kết quả tương tự nhưng kiểm soát linh hoạt hơn đối với file descriptor được dùng. Trong mục đó, ta cũng trình bày ví dụ về lý do tại sao việc kiểm soát file descriptor mà file được mở lại hữu ích.

## **4.3.1 Tham số flags của open()**

<span id="page-53-0"></span>Trong một số ví dụ gọi open() trong [Listing 4-2,](#page-52-1) ta đã bao gồm các bit khác (O\_CREAT, O\_TRUNC và O\_APPEND) trong flags ngoài access mode của file. Bây giờ ta xem xét tham số flags chi tiết hơn. Bảng 4-3 tóm tắt toàn bộ tập hằng số có thể được bitwise OR (|) trong flags. Cột cuối cùng chỉ ra hằng số nào được chuẩn hóa trong SUSv3 hoặc SUSv4.

**Bảng 4-3:** Các giá trị cho tham số flags của open()

| Flag        | Mục đích                                                         | SUS? |
|-------------|------------------------------------------------------------------|------|
| O_RDONLY    | Mở để chỉ đọc                                                    | v3   |
| O_WRONLY    | Mở để chỉ ghi                                                    | v3   |
| O_RDWR      | Mở để đọc và ghi                                                 | v3   |
| O_CLOEXEC   | Đặt close-on-exec flag (từ Linux 2.6.23)                        | v4   |
| O_CREAT     | Tạo file nếu chưa tồn tại                                        | v3   |
| O_DIRECT    | File I/O bỏ qua buffer cache                                     |      |
| O_DIRECTORY | Thất bại nếu pathname không phải directory                       | v4   |
| O_EXCL      | Kết hợp với O_CREAT: tạo file độc quyền                         | v3   |
| O_LARGEFILE | Dùng trên hệ thống 32-bit để mở large file                       |      |
| O_NOATIME   | Không cập nhật thời gian truy cập file khi read() (từ Linux 2.6.8) |   |
| O_NOCTTY    | Không để pathname trở thành controlling terminal                 | v3   |
| O_NOFOLLOW  | Không dereference symbolic link                                  | v4   |
| O_TRUNC     | Truncate file hiện có về độ dài bằng không                       | v3   |
| O_APPEND    | Các lần ghi luôn được append vào cuối file                       | v3   |
| O_ASYNC     | Sinh ra một signal khi I/O có thể thực hiện                      |      |
| O_DSYNC     | Cung cấp synchronized I/O data integrity (từ Linux 2.6.33)      | v3   |
| O_NONBLOCK  | Mở trong nonblocking mode                                        | v3   |
| O_SYNC      | Làm cho các lần ghi file trở thành synchronous                   | v3   |

Các hằng số trong Bảng 4-3 được chia thành các nhóm sau:

- **File access mode flag:** Đây là các flag O\_RDONLY, O\_WRONLY và O\_RDWR đã mô tả trước đó. Chúng có thể được lấy lại bằng thao tác fcntl() F\_GETFL ([Mục 5.3](#page-72-0)).
- **File creation flag:** Đây là các flag hiển thị trong phần thứ hai của Bảng 4-3. Chúng kiểm soát nhiều khía cạnh hành vi của lời gọi open(), cũng như các tùy chọn cho các thao tác I/O tiếp theo. Những flag này không thể lấy lại hay thay đổi.
- **Open file status flag:** Đây là các flag còn lại trong Bảng 4-3. Chúng có thể được lấy lại và sửa đổi bằng các thao tác fcntl() F\_GETFL và F\_SETFL ([Mục 5.3](#page-72-0)). Các flag này đôi khi được gọi đơn giản là file status flag.

Từ kernel 2.6.22, các file dành riêng cho Linux trong thư mục /proc/PID/fdinfo có thể được đọc để lấy thông tin về file descriptor của bất kỳ process nào trên hệ thống. Trong thư mục này có một file ứng với mỗi file descriptor đang mở của process, với tên khớp với số của descriptor đó. Trường pos trong file này hiển thị file offset hiện tại ([Mục 4.7](#page-60-1)). Trường flags là một số octal hiển thị file access mode flag và open file status flag. (Để giải mã số này, ta cần tra cứu các giá trị số của những flag này trong các header file của thư viện C.)

Chi tiết của các hằng số flag như sau:

#### O\_APPEND

Các lần ghi luôn được append vào cuối file. Ta thảo luận về ý nghĩa của flag này trong [Mục 5.1.](#page-69-0)

#### O\_ASYNC

Sinh ra một signal khi I/O trở nên khả thi trên file descriptor được trả về bởi open(). Tính năng này, được gọi là signal-driven I/O, chỉ khả dụng với một số loại file nhất định, chẳng hạn như terminal, FIFO và socket. (Flag O\_ASYNC không được quy định trong SUSv3; tuy nhiên, nó, hoặc tên đồng nghĩa cũ FASYNC, được tìm thấy trên hầu hết các phiên bản UNIX.) Trên Linux, việc chỉ định flag O\_ASYNC khi gọi open() không có hiệu lực. Để kích hoạt signal-driven I/O, ta phải đặt flag này bằng thao tác fcntl() F\_SETFL ([Mục 5.3](#page-72-0)). (Một số phiên bản UNIX khác cũng hoạt động tương tự.) Tham khảo Mục 63.3 để biết thêm thông tin về flag O\_ASYNC.

#### O\_CLOEXEC (từ Linux 2.6.23)

Kích hoạt close-on-exec flag (FD\_CLOEXEC) cho file descriptor mới. Ta mô tả flag FD\_CLOEXEC trong Mục 27.4. Việc sử dụng flag O\_CLOEXEC cho phép chương trình tránh các thao tác fcntl() F\_SETFD và F\_SETFD bổ sung để đặt close-on-exec flag. Nó cũng cần thiết trong các chương trình multithreaded để tránh race condition có thể xảy ra khi dùng kỹ thuật sau này. Những race này có thể xảy ra khi một thread mở một file descriptor rồi cố gắng đánh dấu nó close-on-exec đồng thời với việc một thread khác thực hiện fork() rồi exec() của một chương trình tùy ý. (Giả sử thread thứ hai thực hiện được cả fork() lẫn exec() trong khoảng thời gian giữa lúc thread đầu tiên mở file descriptor và dùng fcntl() để đặt close-on-exec flag.) Những race như vậy có thể dẫn đến việc file descriptor đang mở vô tình được truyền cho các chương trình không an toàn. (Ta đề cập thêm về race condition trong [Mục 5.1](#page-69-0).)

#### O\_CREAT

Nếu file chưa tồn tại, nó được tạo ra là một file mới và rỗng. Flag này có hiệu lực ngay cả khi file chỉ được mở để đọc. Nếu ta chỉ định O\_CREAT, ta phải cung cấp tham số mode trong lời gọi open(); nếu không, permission của file mới sẽ được đặt thành một giá trị ngẫu nhiên từ stack.

#### O\_DIRECT

Cho phép file I/O bỏ qua buffer cache. Tính năng này được mô tả trong Mục 13.6. Feature test macro \_GNU\_SOURCE phải được định nghĩa để hằng số này có thể dùng được từ `<fcntl.h>`.

#### O\_DIRECTORY

Trả về lỗi (errno bằng ENOTDIR) nếu pathname không phải là một directory. Flag này là một extension được thiết kế đặc biệt để triển khai opendir() (Mục 18.8). Feature test macro \_GNU\_SOURCE phải được định nghĩa để hằng số này có thể dùng được từ `<fcntl.h>`.

#### O\_DSYNC (từ Linux 2.6.33)

Thực hiện ghi file theo yêu cầu của synchronized I/O data integrity completion. Xem thảo luận về kernel I/O buffering trong Mục 13.3.

#### O\_EXCL

Flag này được dùng kết hợp với O\_CREAT để chỉ ra rằng nếu file đã tồn tại, nó không nên được mở; thay vào đó, open() phải thất bại với errno được đặt thành EEXIST. Nói cách khác, flag này cho phép caller đảm bảo rằng nó là process đang tạo file. Việc kiểm tra sự tồn tại và tạo file được thực hiện một cách atomic. Ta thảo luận về khái niệm atomicity trong [Mục 5.1](#page-69-0). Khi cả O\_CREAT và O\_EXCL được chỉ định trong flags, open() thất bại (với lỗi EEXIST) nếu pathname là một symbolic link. SUSv3 yêu cầu hành vi này để ứng dụng có đặc quyền có thể tạo file ở một vị trí đã biết mà không có khả năng symbolic link sẽ khiến file được tạo ở vị trí khác (ví dụ như thư mục hệ thống), điều có thể gây ra các vấn đề bảo mật.

#### O\_LARGEFILE

Mở file với hỗ trợ large file. Flag này được dùng trên hệ thống 32-bit để làm việc với large file. Mặc dù không được quy định trong SUSv3, flag O\_LARGEFILE được tìm thấy trên một số phiên bản UNIX khác. Trên các phiên bản Linux 64-bit như Alpha và IA-64, flag này không có hiệu lực. Xem [Mục 5.10](#page-83-0) để biết thêm thông tin.

#### O\_NOATIME (từ Linux 2.6.8)

Không cập nhật thời gian truy cập file lần cuối (trường st\_atime được mô tả trong Mục 15.1) khi đọc từ file này. Để sử dụng flag này, effective user ID của process gọi phải khớp với owner của file, hoặc process phải có đặc quyền (CAP\_FOWNER); nếu không, open() thất bại với lỗi EPERM. (Trong thực tế, đối với process không có đặc quyền, filesystem user ID của process, chứ không phải effective user ID, mới phải khớp với user ID của file khi mở file với flag O\_NOATIME, như được mô tả trong Mục 9.5.) Flag này là một Linux extension phi chuẩn. Để hiển thị định nghĩa của nó từ `<fcntl.h>`, ta phải định nghĩa feature test macro \_GNU\_SOURCE. Flag O\_NOATIME được thiết kế dùng cho các chương trình indexing và backup. Việc sử dụng nó có thể giảm đáng kể lượng hoạt động của disk, vì không cần thiết phải đọc tới đọc lui trên disk để đọc nội dung file và cập nhật thời gian truy cập lần cuối trong i-node của file (Mục 14.4). Chức năng tương tự với O\_NOATIME có thể đạt được bằng flag MS\_NOATIME của mount() (Mục 14.8.1) và flag FS\_NOATIME\_FL (Mục 15.5).

#### O\_NOCTTY

Nếu file đang được mở là một terminal device, hãy ngăn nó trở thành controlling terminal. Controlling terminal được thảo luận trong Mục 34.4. Nếu file đang được mở không phải là terminal, flag này không có hiệu lực.

#### O\_NOFOLLOW

Thông thường, open() dereferences pathname nếu nó là một symbolic link. Tuy nhiên, nếu flag O\_NOFOLLOW được chỉ định, open() thất bại (với errno được đặt thành ELOOP) nếu pathname là một symbolic link. Flag này hữu ích, đặc biệt trong các chương trình có đặc quyền, để đảm bảo rằng open() không dereference một symbolic link. Để hiển thị định nghĩa của flag này từ `<fcntl.h>`, ta phải định nghĩa feature test macro \_GNU\_SOURCE.

#### O\_NONBLOCK

Mở file trong nonblocking mode. Xem [Mục 5.9](#page-82-0).

#### O\_SYNC

Mở file cho synchronous I/O. Xem thảo luận về kernel I/O buffering trong Mục 13.3.

#### O\_TRUNC

Nếu file đã tồn tại và là một regular file, hãy truncate nó về độ dài bằng không, hủy bỏ mọi dữ liệu hiện có. Trên Linux, việc truncation xảy ra bất kể file được mở để đọc hay ghi (trong cả hai trường hợp, ta phải có write permission trên file). SUSv3 để ngỏ tổ hợp O\_RDONLY và O\_TRUNC, nhưng hầu hết các phiên bản UNIX khác đều hoạt động giống Linux.

## **4.3.2 Các lỗi từ open()**

Nếu xảy ra lỗi khi cố gắng mở file, open() trả về –1, và errno xác định nguyên nhân lỗi. Sau đây là một số lỗi có thể xảy ra (ngoài những lỗi đã được lưu ý khi mô tả tham số flags ở trên):

#### EACCES

Permission của file không cho phép process gọi mở file ở mode được chỉ định bởi flags. Hoặc, do permission của thư mục, file không thể được truy cập, hoặc file không tồn tại và không thể được tạo.

#### EISDIR

File được chỉ định là một directory, và caller cố gắng mở nó để ghi. Điều này không được phép. (Mặt khác, có những trường hợp việc mở directory để đọc lại hữu ích. Ta xem xét một ví dụ trong Mục 18.11.)

#### EMFILE

Giới hạn tài nguyên của process về số lượng open file descriptor đã đạt tới (RLIMIT\_NOFILE, được mô tả trong Mục 36.3).

#### ENFILE

Giới hạn toàn hệ thống về số lượng file đang mở đã đạt tới.

#### ENOENT

File được chỉ định không tồn tại, và O\_CREAT không được chỉ định, hoặc O\_CREAT được chỉ định, và một trong các thư mục trong pathname không tồn tại hoặc là một symbolic link trỏ đến pathname không tồn tại (dangling link).

#### EROFS

File được chỉ định nằm trên một read-only file system và caller cố gắng mở nó để ghi.

#### ETXTBSY

File được chỉ định là một executable file (chương trình) đang được thực thi. Không được phép sửa đổi (tức là mở để ghi) executable file liên kết với chương trình đang chạy. (Ta phải kết thúc chương trình trước mới có thể sửa đổi executable file.)

Khi mô tả các system call hoặc library function khác sau này, ta thường sẽ không liệt kê chi tiết các lỗi có thể xảy ra như ở trên. (Danh sách đó có thể tìm thấy trong trang manual tương ứng cho mỗi system call hoặc library function.) Ta làm như vậy ở đây vì hai lý do. Một là open() là system call đầu tiên ta mô tả chi tiết, và danh sách trên minh họa rằng một system call hay library function có thể thất bại vì bất kỳ lý do nào trong số nhiều lý do. Thứ hai, các lý do cụ thể khiến open() có thể thất bại tự chúng đã là một danh sách thú vị, minh họa một số yếu tố và kiểm tra xuất hiện khi một file được truy cập. (Danh sách trên chưa đầy đủ: xem trang manual open(2) để biết thêm lý do tại sao open() có thể thất bại.)

# **4.3.3 System Call creat()**

Trong các phiên bản UNIX đời đầu, open() chỉ có hai tham số và không thể dùng để tạo file mới. Thay vào đó, system call creat() được dùng để tạo và mở một file mới.

```
#include <fcntl.h>
int creat(const char *pathname, mode_t mode);
                                          Returns file descriptor, or –1 on error
```

System call creat() tạo và mở một file mới với pathname đã cho, hoặc nếu file đã tồn tại, mở file đó và truncate nó về độ dài bằng không. Là kết quả hàm, creat() trả về một file descriptor có thể được dùng trong các system call tiếp theo. Gọi creat() tương đương với lời gọi open() sau đây:

```
fd = open(pathname, O_WRONLY | O_CREAT | O_TRUNC, mode);
```

Vì tham số flags của open() cung cấp khả năng kiểm soát tốt hơn về cách file được mở (ví dụ, ta có thể chỉ định O\_RDWR thay vì O\_WRONLY), creat() hiện đã lỗi thời, mặc dù nó vẫn có thể thấy trong các chương trình cũ.

# **4.4 Đọc từ File: read()**

System call read() đọc dữ liệu từ file đang mở được tham chiếu bởi descriptor fd.

```
#include <unistd.h>
ssize_t read(int fd, void *buffer, size_t count);
                        Returns number of bytes read, 0 on EOF, or –1 on error
```

Tham số count chỉ định số byte tối đa để đọc. (Kiểu dữ liệu size\_t là một unsigned integer type.) Tham số buffer cung cấp địa chỉ của memory buffer mà dữ liệu đầu vào sẽ được đặt vào. Buffer này phải có độ dài ít nhất count byte.

> System call không cấp phát memory cho các buffer được dùng để trả thông tin về cho caller. Thay vào đó, ta phải truyền vào một pointer đến memory buffer đã được cấp phát trước với kích thước phù hợp. Điều này trái ngược với một số library function có cấp phát memory buffer để trả thông tin về cho caller.

Một lời gọi read() thành công trả về số byte thực sự đã đọc, hoặc 0 nếu gặp end-of-file. Khi có lỗi, giá trị thông thường –1 được trả về. Kiểu dữ liệu ssize\_t là một signed integer type dùng để chứa số byte hoặc chỉ báo lỗi –1.

Một lời gọi read() có thể đọc ít hơn số byte được yêu cầu. Đối với regular file, nguyên nhân có thể là do ta đang gần cuối file.

Khi read() được áp dụng cho các loại file khác — như pipe, FIFO, socket hay terminal — cũng có nhiều trường hợp khác nhau mà nó có thể đọc ít byte hơn yêu cầu. Ví dụ, theo mặc định, một read() từ terminal chỉ đọc ký tự đến ký tự newline (\n) tiếp theo. Ta xem xét các trường hợp này khi đề cập đến các loại file khác trong các chương sau.

Khi dùng read() để đọc một chuỗi ký tự từ, chẳng hạn như terminal, ta có thể kỳ vọng đoạn code sau hoạt động được:

```
#define MAX_READ 20
char buffer[MAX_READ];
if (read(STDIN_FILENO, buffer, MAX_READ) == -1)
 errExit("read");
printf("The input data was: %s\n", buffer);
```

Kết quả đầu ra từ đoạn code này có thể sẽ lạ, vì nó có thể bao gồm các ký tự ngoài chuỗi thực sự được nhập vào. Điều này xảy ra bởi vì read() không đặt null byte kết thúc vào cuối chuỗi mà printf() được yêu cầu in ra. Suy nghĩ một chút sẽ khiến ta nhận ra điều này phải như vậy, vì read() có thể được dùng để đọc bất kỳ chuỗi byte nào từ file. Trong một số trường hợp, đầu vào này có thể là text, nhưng trong các trường hợp khác, đầu vào có thể là các số nguyên nhị phân hoặc các C structure ở dạng nhị phân. read() không có cách nào phân biệt được, và vì vậy nó không thể tuân theo quy ước C về null terminating character string. Nếu cần null byte kết thúc ở cuối input buffer, ta phải tự đặt vào:

```
char buffer[MAX_READ + 1];
ssize_t numRead;
numRead = read(STDIN_FILENO, buffer, MAX_READ);
if (numRead == -1)
 errExit("read");
buffer[numRead] = '\0';
printf("The input data was: %s\n", buffer);
```

Vì null byte kết thúc cần một byte bộ nhớ, kích thước của buffer phải ít nhất lớn hơn một so với chuỗi dài nhất ta muốn đọc.

# **4.5 Ghi vào File: write()**

System call write() ghi dữ liệu vào một file đang mở.

```
#include <unistd.h>
ssize_t write(int fd, void *buffer, size_t count);
                                Returns number of bytes written, or –1 on error
```

Các tham số của write() tương tự như của read(): buffer là địa chỉ của dữ liệu cần ghi; count là số byte cần ghi từ buffer; và fd là file descriptor tham chiếu đến file mà dữ liệu sẽ được ghi vào.

Khi thành công, write() trả về số byte thực sự đã ghi; con số này có thể nhỏ hơn count. Đối với disk file, các lý do có thể cho việc partial write như vậy là disk đã đầy hoặc giới hạn tài nguyên của process về kích thước file đã đạt tới. (Giới hạn liên quan là RLIMIT\_FSIZE, được mô tả trong Mục 36.3.)

Khi thực hiện I/O trên disk file, một lần trả về thành công từ write() không đảm bảo rằng dữ liệu đã được chuyển đến disk, vì kernel thực hiện buffering I/O disk để giảm hoạt động disk và tăng tốc độ các lời gọi write(). Ta xem xét chi tiết này trong Chương 13.

# **4.6 Đóng File: close()**

System call close() đóng một open file descriptor, giải phóng nó để process tái sử dụng sau này. Khi một process kết thúc, tất cả các open file descriptor của nó tự động được đóng.

```
#include <unistd.h>
int close(int fd);
                                              Returns 0 on success, or –1 on error
```

Thông thường nên thực hành tốt là đóng các file descriptor không cần thiết một cách tường minh, vì điều này làm cho code dễ đọc và đáng tin cậy hơn trước các sửa đổi tiếp theo. Hơn nữa, file descriptor là tài nguyên có thể cạn kiệt, vì vậy việc không đóng file descriptor có thể dẫn đến một process hết descriptor. Đây là vấn đề đặc biệt quan trọng khi viết các chương trình chạy lâu dài xử lý nhiều file, chẳng hạn như shell hoặc network server.

Cũng như mọi system call khác, một lời gọi close() nên được bọc trong code kiểm tra lỗi, chẳng hạn như sau:

```
if (close(fd) == -1)
 errExit("close");
```

Điều này phát hiện các lỗi như cố gắng đóng file descriptor chưa mở hoặc đóng cùng một file descriptor hai lần, và phát hiện các điều kiện lỗi mà một file system cụ thể có thể chẩn đoán trong một thao tác close.

> NFS (Network File System) cung cấp ví dụ về lỗi đặc thù của một file system. Nếu xảy ra NFS commit failure, nghĩa là dữ liệu không đến được remote disk, thì lỗi này được truyền đến ứng dụng dưới dạng thất bại trong lời gọi close().

## <span id="page-60-1"></span>**4.7 Thay đổi File Offset: lseek()**

<span id="page-60-0"></span>Đối với mỗi file đang mở, kernel ghi lại một file offset, đôi khi còn được gọi là read-write offset hoặc pointer. Đây là vị trí trong file mà lần read() hoặc write() tiếp theo sẽ bắt đầu. File offset được biểu thị bằng vị trí byte thứ tự so với đầu file. Byte đầu tiên của file ở offset 0.

File offset được đặt để trỏ đến đầu file khi file được mở và tự động được điều chỉnh bởi mỗi lời gọi read() hoặc write() tiếp theo để nó trỏ đến byte tiếp theo của file sau các byte vừa được đọc hoặc ghi. Do đó, các lần gọi read() và write() liên tiếp tiến dần qua file một cách tuần tự.

System call lseek() điều chỉnh file offset của file đang mở được tham chiếu bởi file descriptor fd, theo các giá trị được chỉ định trong offset và whence.

```
#include <unistd.h>
off_t lseek(int fd, off_t offset, int whence);
                               Returns new file offset if successful, or –1 on error
```

Tham số offset chỉ định một giá trị tính bằng byte. (Kiểu dữ liệu off\_t là một signed integer type được SUSv3 quy định.) Tham số whence chỉ ra điểm gốc từ đó offset được diễn giải, và là một trong các giá trị sau:

**SEEK\_SET**

File offset được đặt cách đầu file offset byte.

**SEEK\_CUR**

File offset được điều chỉnh offset byte so với file offset hiện tại.

**SEEK\_END**

File offset được đặt bằng kích thước file cộng với offset. Nói cách khác, offset được diễn giải so với byte tiếp theo sau byte cuối cùng của file.

Hình 4-1 cho thấy cách tham số whence được diễn giải.

Trong các phiên bản UNIX đời đầu, người ta dùng các số nguyên 0, 1 và 2 thay vì các hằng số SEEK\_\* hiển thị trong phần chính. Các phiên bản BSD cũ hơn dùng các tên khác nhau cho các giá trị này: L\_SET, L\_INCR và L\_XTND.

```txt
                      File chứa N byte dữ liệu
        <-------------------------------------------------------->

số byte     0     1     ...     ...     N-2     N-1     N     N+1
           +-----+-----+-----+-----+-----+-----+-----+-----+-----+
           |  0  |  1  | ... | ... | N-2 | N-1 |     |     |     |
           +-----+-----+-----+-----+-----+-----+-----+-----+-----+
                                                  ^     ^
                                                  |     |
                                           Các byte chưa ghi
                                               sau EOF

                              ^
                              |
                       File offset hiện tại
```

**Hình 4-1:** Diễn giải tham số whence của lseek()

Nếu whence là SEEK\_CUR hoặc SEEK\_END, offset có thể âm hoặc dương; đối với SEEK\_SET, offset phải không âm.

Giá trị trả về từ một lseek() thành công là file offset mới. Lời gọi sau đây lấy vị trí hiện tại của file offset mà không thay đổi nó:

```
curr = lseek(fd, 0, SEEK_CUR);
```

Một số phiên bản UNIX (nhưng không phải Linux) có hàm phi chuẩn tell(fd), phục vụ cùng mục đích như lời gọi lseek() ở trên.

Đây là một số ví dụ về các lời gọi lseek() khác, cùng với comment chỉ ra nơi file offset được di chuyển đến:

```
lseek(fd, 0, SEEK_SET);    /* Đầu file */
lseek(fd, 0, SEEK_END);    /* Byte tiếp theo sau cuối file */
lseek(fd, -1, SEEK_END);   /* Byte cuối cùng của file */
lseek(fd, -10, SEEK_CUR);  /* Mười byte trước vị trí hiện tại */
lseek(fd, 10000, SEEK_END); /* 10001 byte sau byte cuối cùng của file */
```

Việc gọi lseek() chỉ đơn giản điều chỉnh bản ghi file offset của kernel liên kết với một file descriptor. Nó không gây ra bất kỳ truy cập physical device nào.

Ta mô tả thêm một số chi tiết về mối quan hệ giữa file offset, file descriptor và open file trong Mục [5.4](#page-73-0).

Ta không thể áp dụng lseek() cho tất cả các loại file. Áp dụng lseek() cho pipe, FIFO, socket hay terminal không được phép; lseek() thất bại, với errno được đặt thành ESPIPE. Mặt khác, có thể áp dụng lseek() cho các device mà ở đó điều đó có ý nghĩa. Ví dụ, có thể seek đến một vị trí cụ thể trên disk hoặc tape device.

> Chữ l trong tên lseek() có nguồn gốc từ thực tế là tham số offset và giá trị trả về ban đầu đều được định kiểu là long. Các phiên bản UNIX đời đầu cung cấp system call seek(), định kiểu các giá trị này là int.

#### **File hole**

Điều gì xảy ra nếu một chương trình seek qua cuối file, rồi thực hiện I/O? Một lời gọi read() sẽ trả về 0, báo hiệu end-of-file. Đáng ngạc nhiên là có thể ghi byte vào một điểm tùy ý sau cuối file.

Khoảng trống giữa cuối file cũ và các byte mới được ghi được gọi là file hole. Từ góc độ lập trình, các byte trong hole tồn tại, và đọc từ hole sẽ trả về một buffer chứa các byte 0 (null byte).

Tuy nhiên, file hole không chiếm dung lượng disk. File system không cấp phát bất kỳ disk block nào cho một hole cho đến khi, tại một thời điểm sau đó, dữ liệu được ghi vào đó. Ưu điểm chính của file hole là một file thưa thớt sẽ tiêu thụ ít dung lượng disk hơn so với yêu cầu nếu các null byte thực sự cần được cấp phát trong các disk block. Core dump file (Mục 22.1) là ví dụ phổ biến về các file chứa nhiều hole lớn.

> Nhận định rằng file hole không tiêu thụ dung lượng disk cần được bổ sung đôi chút. Trên hầu hết các file system, không gian file được cấp phát theo đơn vị block (Mục 14.3). Kích thước của một block phụ thuộc vào file system, nhưng thường là khoảng 1024, 2048 hoặc 4096 byte. Nếu ranh giới của một hole nằm trong một block, thay vì tại ranh giới block, thì một block đầy đủ sẽ được cấp phát để lưu trữ dữ liệu trong phần còn lại của block, và phần tương ứng với hole được điền bằng null byte.

Hầu hết các native UNIX file system đều hỗ trợ khái niệm file hole, nhưng nhiều nonnative file system (ví dụ: VFAT của Microsoft) thì không. Trên một file system không hỗ trợ hole, các null byte rõ ràng sẽ được ghi vào file.

Sự tồn tại của hole có nghĩa là kích thước danh nghĩa của file có thể lớn hơn lượng disk storage mà nó sử dụng (trong một số trường hợp, lớn hơn đáng kể). Ghi byte vào giữa file hole sẽ làm giảm dung lượng disk trống khi kernel cấp phát block để lấp đầy hole, ngay cả khi kích thước file không thay đổi. Tình huống như vậy không phổ biến, nhưng vẫn nên lưu ý.

> SUSv3 quy định một hàm, posix\_fallocate(fd, offset, len), đảm bảo rằng không gian được cấp phát trên disk cho phạm vi byte được chỉ định bởi offset và len cho disk file được tham chiếu bởi descriptor fd. Điều này cho phép ứng dụng chắc chắn rằng một write() sau này vào file sẽ không thất bại vì hết dung lượng disk (điều có thể xảy ra nếu hole trong file bị lấp đầy, hoặc một ứng dụng khác tiêu thụ dung lượng trên disk). Trước đây, phiên bản glibc của hàm này đạt được kết quả mong muốn bằng cách ghi một byte 0 vào mỗi block trong phạm vi đã chỉ định. Từ phiên bản 2.6.23, Linux cung cấp system call fallocate(), mang lại cách hiệu quả hơn để đảm bảo rằng dung lượng cần thiết được cấp phát, và phiên bản glibc của posix\_fallocate() sử dụng system call này khi khả dụng.

Mục 14.4 mô tả cách các hole được biểu diễn trong một file, và Mục 15.1 mô tả system call stat(), có thể cho biết kích thước hiện tại của file, cũng như số block thực sự được cấp phát cho file.

#### **Chương trình ví dụ**

[Listing 4-3](#page-63-0) minh họa việc sử dụng lseek() kết hợp với read() và write(). Đối số command-line đầu tiên của chương trình này là tên file cần mở. Các đối số còn lại chỉ định các thao tác I/O cần thực hiện trên file. Mỗi thao tác này bao gồm một chữ cái theo sau là một giá trị liên quan (không có khoảng cách phân cách):

- `soffset`: Seek đến byte offset tính từ đầu file.
- `rlength`: Đọc length byte từ file, bắt đầu từ file offset hiện tại, và hiển thị chúng ở dạng text.
- `Rlength`: Đọc length byte từ file, bắt đầu từ file offset hiện tại, và hiển thị chúng ở dạng hexadecimal.
- `wstr`: Ghi chuỗi ký tự được chỉ định trong str tại file offset hiện tại.

<span id="page-63-0"></span>**Listing 4-3:** Minh họa read(), write() và lseek()

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

Phiên log shell sau đây minh họa cách sử dụng chương trình trong Listing [4-3,](#page-63-0) cho thấy điều gì xảy ra khi ta cố gắng đọc byte từ một file hole:

```
$ touch tfile                         Tạo file mới, rỗng
$ ./seek_io tfile s100000 wabc        Seek đến offset 100.000, ghi "abc"
s100000: seek succeeded
wabc: wrote 3 bytes
```

\$ **ls -l tfile** Kiểm tra kích thước file -rw-r--r-- 1 mtk users 100003 Feb 10 10:35 tfile s10000: seek succeeded

\$ **./seek\_io tfile s10000 R5** Seek đến offset 10.000, đọc 5 byte từ hole

R5: 00 00 00 00 00 Các byte trong hole chứa giá trị 0

# <span id="page-65-0"></span>**4.8 Các thao tác nằm ngoài mô hình I/O phổ quát: ioctl()**

System call ioctl() là một cơ chế đa năng để thực hiện các thao tác file và device nằm ngoài mô hình I/O phổ quát được mô tả trước đó trong chương này.

```
#include <sys/ioctl.h>
int ioctl(int fd, int request, ... /* argp */);
                   Value returned on success depends on request, or –1 on error
```

Tham số fd là một open file descriptor cho device hoặc file mà thao tác điều khiển được chỉ định bởi request sẽ được thực hiện. Các header file đặc thù của device định nghĩa các hằng số có thể được truyền vào tham số request.

Như được chỉ ra bởi ký hiệu dấu chấm lửng (...) chuẩn C, tham số thứ ba của ioctl(), mà ta gọi là argp, có thể có bất kỳ kiểu nào. Giá trị của tham số request cho phép ioctl() xác định loại giá trị cần mong đợi trong argp. Thông thường, argp là một pointer đến một số nguyên hoặc một structure; trong một số trường hợp, nó không được sử dụng.

Ta sẽ thấy một số cách dùng cho ioctl() trong các chương sau (xem ví dụ, Mục 15.5).

> Đặc tả duy nhất mà SUSv3 đưa ra cho ioctl() là cho các thao tác điều khiển STREAMS device. (Tiện ích STREAMS là một tính năng của System V không được hỗ trợ bởi mainline Linux kernel, mặc dù một số phiên bản add-on đã được phát triển.) Không có thao tác ioctl() nào khác được mô tả trong cuốn sách này được quy định trong SUSv3. Tuy nhiên, lời gọi ioctl() đã là một phần của hệ thống UNIX từ các phiên bản đầu tiên, và do đó một số thao tác ioctl() mà ta mô tả được cung cấp trên nhiều phiên bản UNIX khác. Khi mô tả mỗi thao tác ioctl(), ta sẽ ghi chú về các vấn đề tính di động.

# **4.9 Tóm tắt**

Để thực hiện I/O trên regular file, trước tiên ta phải lấy file descriptor bằng open(). Sau đó I/O được thực hiện bằng read() và write(). Sau khi hoàn thành tất cả I/O, ta nên giải phóng file descriptor và các tài nguyên liên quan bằng close(). Các system call này có thể được dùng để thực hiện I/O trên tất cả các loại file.

Thực tế là tất cả các loại file và device driver đều triển khai cùng một giao diện I/O tạo nên tính phổ quát của I/O, có nghĩa là một chương trình thường có thể được dùng với bất kỳ loại file nào mà không cần code đặc thù cho loại file đó.

Đối với mỗi file đang mở, kernel duy trì một file offset, xác định vị trí mà lần read hoặc write tiếp theo sẽ xảy ra. File offset được cập nhật ngầm định bởi các lần read và write. Sử dụng lseek(), ta có thể tường minh đặt lại file offset đến bất kỳ vị trí nào trong file hoặc sau cuối file. Ghi dữ liệu tại một vị trí sau cuối file trước đó sẽ tạo ra một hole trong file. Đọc từ một file hole sẽ trả về các byte chứa giá trị không.

System call ioctl() là giải pháp tổng hợp cho các thao tác device và file không phù hợp với mô hình file I/O chuẩn.

## **4.10 Bài tập**

- **4-1.** Lệnh tee đọc standard input của nó cho đến end-of-file, ghi một bản sao của đầu vào ra standard output và vào file được đặt tên trong đối số command-line của nó. (Ta trình bày một ví dụ về cách dùng lệnh này khi ta thảo luận về FIFO trong Mục 44.7.) Hãy triển khai tee bằng các I/O system call. Theo mặc định, tee ghi đè bất kỳ file hiện có nào có tên đã cho. Hãy triển khai tùy chọn command-line –a (tee –a file), khiến tee append text vào cuối file nếu nó đã tồn tại. (Tham khảo Phụ lục B để biết mô tả về hàm getopt(), có thể được dùng để phân tích cú pháp command-line option.)
- **4-2.** Viết một chương trình giống cp mà, khi được dùng để sao chép một regular file chứa hole (chuỗi null byte), cũng tạo ra các hole tương ứng trong file đích.
