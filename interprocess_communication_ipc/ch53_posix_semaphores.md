## Chương 53
# POSIX SEMAPHORES

Chương này mô tả POSIX semaphores, cho phép các process và thread đồng bộ hóa việc truy cập vào các tài nguyên dùng chung. Trong Chương 47, chúng ta đã mô tả System V semaphores, và chúng ta sẽ giả sử rằng người đọc đã quen thuộc với các khái niệm chung về semaphore và lý do sử dụng semaphore đã được trình bày ở đầu chương đó. Trong suốt chương này, chúng ta sẽ so sánh giữa POSIX semaphores và System V semaphores để làm rõ những điểm giống nhau và khác nhau giữa hai API semaphore này.

## **53.1 Tổng quan**

SUSv3 quy định hai loại POSIX semaphores:

- Named semaphores (semaphore có tên): Loại semaphore này có một tên. Bằng cách gọi `sem_open()` với cùng một tên, các process không liên quan có thể truy cập cùng một semaphore.
- Unnamed semaphores (semaphore không có tên): Loại semaphore này không có tên; thay vào đó, nó nằm ở một vị trí đã thỏa thuận trong bộ nhớ. Unnamed semaphores có thể được chia sẻ giữa các process hoặc giữa một nhóm thread. Khi chia sẻ giữa các process, semaphore phải nằm trong một vùng shared memory (System V, POSIX, hoặc `mmap()`). Khi chia sẻ giữa các thread, semaphore có thể nằm trong một vùng bộ nhớ được chia sẻ bởi các thread (ví dụ: trên heap hoặc trong một biến global).

POSIX semaphores hoạt động theo cách tương tự như System V semaphores; tức là, một POSIX semaphore là một số nguyên mà giá trị của nó không được phép giảm xuống dưới 0. Nếu một process cố giảm giá trị của một semaphore xuống dưới 0, thì tùy thuộc vào hàm được sử dụng, lời gọi sẽ hoặc bị block hoặc thất bại với một lỗi cho biết rằng thao tác hiện tại không thể thực hiện được.

Một số hệ thống không cung cấp đầy đủ hiện thực của POSIX semaphores. Một hạn chế điển hình là chỉ hỗ trợ unnamed thread-shared semaphores. Đó là tình trạng trên Linux 2.4; với Linux 2.6 và glibc cung cấp NPTL, một hiện thực đầy đủ của POSIX semaphores đã có sẵn.

> Trên Linux 2.6 với NPTL, các thao tác semaphore (tăng và giảm) được hiện thực bằng cách sử dụng system call `futex(2)`.

## **53.2 Named Semaphores**

Để làm việc với một named semaphore, chúng ta sử dụng các hàm sau:

- Hàm `sem_open()` mở hoặc tạo một semaphore, khởi tạo semaphore nếu nó được tạo bởi lời gọi, và trả về một handle để sử dụng trong các lời gọi tiếp theo.
- Các hàm `sem_post(sem)` và `sem_wait(sem)` lần lượt tăng và giảm giá trị của một semaphore.
- Hàm `sem_getvalue()` lấy giá trị hiện tại của một semaphore.
- Hàm `sem_close()` xóa bỏ sự liên kết của process đang gọi với một semaphore mà nó đã mở trước đó.
- Hàm `sem_unlink()` xóa tên của một semaphore và đánh dấu semaphore để bị hủy khi tất cả các process đã đóng nó.

SUSv3 không quy định cách hiện thực named semaphores. Một số hiện thực UNIX tạo chúng dưới dạng các file ở một vị trí đặc biệt trong hệ thống file tiêu chuẩn. Trên Linux, chúng được tạo dưới dạng các POSIX shared memory object nhỏ với tên có dạng `sem.name`, trong một hệ thống file tmpfs chuyên dụng (Mục 14.10) được mount dưới thư mục `/dev/shm`. Hệ thống file này có kernel persistence — các semaphore object mà nó chứa sẽ tồn tại ngay cả khi không có process nào đang mở chúng, nhưng chúng sẽ bị mất nếu hệ thống bị tắt.

Named semaphores được hỗ trợ trên Linux kể từ kernel 2.6.

## **53.2.1 Mở một Named Semaphore**

Hàm `sem_open()` tạo và mở một named semaphore mới hoặc mở một semaphore hiện có.

```
#include <fcntl.h> /* Defines O_* constants */
#include <sys/stat.h> /* Defines mode constants */
#include <semaphore.h>
sem_t *sem_open(const char *name, int oflag, ...
 /* mode_t mode, unsigned int value */ );
              Returns pointer to semaphore on success, or SEM_FAILED on error
```

Đối số `name` xác định semaphore. Nó được chỉ định theo các quy tắc được đưa ra trong Mục 51.1.

Đối số `oflag` là một bitmask xác định chúng ta đang mở một semaphore hiện có hay tạo và mở một semaphore mới. Nếu `oflag` là 0, chúng ta đang truy cập một semaphore hiện có. Nếu `O_CREAT` được chỉ định trong `oflag`, thì một semaphore mới được tạo nếu chưa có semaphore nào với tên đã cho. Nếu `oflag` chỉ định cả `O_CREAT` và `O_EXCL`, và một semaphore với tên đã cho đã tồn tại, thì `sem_open()` thất bại.

Nếu `sem_open()` được sử dụng để mở một semaphore hiện có, lời gọi chỉ cần hai đối số. Tuy nhiên, nếu `O_CREAT` được chỉ định trong `flags`, thì cần thêm hai đối số: `mode` và `value`. (Nếu semaphore được chỉ định bởi `name` đã tồn tại, thì hai đối số này bị bỏ qua.) Các đối số này như sau:

- Đối số `mode` là một bitmask chỉ định các quyền được đặt trên semaphore mới. Các giá trị bit giống như đối với file (Bảng 15-4, trang 295), và, như với `open()`, giá trị trong `mode` được mask với process umask (Mục 15.4.6). SUSv3 không chỉ định bất kỳ flag chế độ truy cập nào (`O_RDONLY`, `O_WRONLY`, và `O_RDWR`) cho `oflag`. Nhiều hiện thực, bao gồm Linux, giả định một chế độ truy cập là `O_RDWR` khi mở một semaphore, vì hầu hết các ứng dụng sử dụng semaphore phải sử dụng cả `sem_post()` và `sem_wait()`, liên quan đến việc đọc và sửa đổi giá trị của một semaphore. Điều này có nghĩa là chúng ta nên đảm bảo rằng cả quyền đọc và ghi đều được cấp cho mỗi loại người dùng (owner, group, và other) cần truy cập vào semaphore.
- Đối số `value` là một số nguyên không dấu chỉ định giá trị ban đầu được gán cho semaphore mới. Việc tạo và khởi tạo semaphore được thực hiện theo cách atomic. Điều này tránh được những phức tạp cần thiết cho việc khởi tạo System V semaphores (Mục 47.5).

Bất kể chúng ta đang tạo một semaphore mới hay mở một semaphore hiện có, `sem_open()` trả về một pointer đến một giá trị `sem_t`, và chúng ta sử dụng pointer này trong các lời gọi tiếp theo đến các hàm thao tác trên semaphore. Khi có lỗi, `sem_open()` trả về giá trị `SEM_FAILED`. (Trên hầu hết các hiện thực, `SEM_FAILED` được định nghĩa là `((sem_t *) 0)` hoặc `((sem_t *) –1)`; Linux định nghĩa nó là cái trước.)

SUSv3 nêu rằng kết quả là không xác định nếu chúng ta cố thực hiện các thao tác (`sem_post()`, `sem_wait()`, v.v.) trên một bản sao của biến `sem_t` được trỏ tới bởi giá trị trả về của `sem_open()`. Nói cách khác, cách sử dụng `sem2` sau đây là không được phép:

```
sem_t *sp, sem2
sp = sem_open(...);
sem2 = *sp;
sem_wait(&sem2);
```

Khi một child được tạo thông qua `fork()`, nó kế thừa các tham chiếu đến tất cả các named semaphores đang mở trong process cha của nó. Sau `fork()`, process cha và process con có thể sử dụng các semaphore này để đồng bộ hóa các hành động của chúng.

#### **Chương trình ví dụ**

Chương trình trong Listing 53-1 cung cấp một giao diện dòng lệnh đơn giản cho hàm `sem_open()`. Định dạng lệnh cho chương trình này được hiển thị trong hàm `usageError()`.

Phiên log shell sau đây minh họa việc sử dụng chương trình này. Đầu tiên chúng ta sử dụng lệnh `umask` để từ chối tất cả các quyền cho người dùng thuộc class `other`. Sau đó chúng ta tạo độc quyền một semaphore và kiểm tra nội dung của thư mục ảo dành riêng cho Linux chứa các named semaphores.

```
$ umask 007
$ ./psem_create -cx /demo 666 666 means read+write for all users
$ ls -l /dev/shm/sem.*
-rw-rw---- 1 mtk users 16 Jul 6 12:09 /dev/shm/sem.demo
```

Kết quả đầu ra của lệnh `ls` cho thấy rằng process umask đã ghi đè lên các quyền được chỉ định là đọc cộng ghi cho class người dùng `other`.

Nếu chúng ta thử một lần nữa để tạo độc quyền một semaphore với cùng tên, thao tác thất bại, vì tên đó đã tồn tại.

```
$ ./psem_create -cx /demo 666
ERROR [EEXIST File exists] sem_open Failed because of O_EXCL
```

**Listing 53-1:** Sử dụng `sem_open()` để mở hoặc tạo một POSIX named semaphore

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_create.c
#include <semaphore.h>
#include <sys/stat.h>
#include <fcntl.h>
#include "tlpi_hdr.h"
static void
usageError(const char *progName)
{
 fprintf(stderr, "Usage: %s [-cx] name [octal-perms [value]]\n", progName);
 fprintf(stderr, " -c Create semaphore (O_CREAT)\n");
 fprintf(stderr, " -x Create exclusively (O_EXCL)\n");
 exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
 int flags, opt;
 mode_t perms;
 unsigned int value;
 sem_t *sem;
 flags = 0;
 while ((opt = getopt(argc, argv, "cx")) != -1) {
 switch (opt) {
 case 'c': flags |= O_CREAT; break;
 case 'x': flags |= O_EXCL; break;
 default: usageError(argv[0]);
 }
 }
```

```
 if (optind >= argc)
 usageError(argv[0]);
 /* Default permissions are rw-------; default semaphore initialization
 value is 0 */
 perms = (argc <= optind + 1) ? (S_IRUSR | S_IWUSR) :
 getInt(argv[optind + 1], GN_BASE_8, "octal-perms");
 value = (argc <= optind + 2) ? 0 : getInt(argv[optind + 2], 0, "value");
 sem = sem_open(argv[optind], flags, perms, value);
 if (sem == SEM_FAILED)
 errExit("sem_open");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_create.c
```

## **53.2.2 Đóng một Semaphore**

Khi một process mở một named semaphore, hệ thống ghi lại sự liên kết giữa process và semaphore. Hàm `sem_close()` kết thúc sự liên kết này (tức là đóng semaphore), giải phóng bất kỳ tài nguyên nào mà hệ thống đã liên kết với semaphore cho process này, và giảm số lượng process tham chiếu đến semaphore.

```
#include <semaphore.h>
int sem_close(sem_t *sem);
                                             Returns 0 on success, or –1 on error
```

Các named semaphores đang mở cũng được tự động đóng khi process kết thúc hoặc nếu process thực hiện `exec()`.

Đóng một semaphore không xóa nó. Để thực hiện điều đó, chúng ta cần sử dụng `sem_unlink()`.

## **53.2.3 Xóa một Named Semaphore**

Hàm `sem_unlink()` xóa semaphore được xác định bởi `name` và đánh dấu semaphore để bị hủy sau khi tất cả các process ngừng sử dụng nó (điều này có thể có nghĩa là ngay lập tức, nếu tất cả các process đã mở semaphore đã đóng nó).

```
#include <semaphore.h>
int sem_unlink(const char *name);
                                            Returns 0 on success, or –1 on error
```

Listing 53-2 minh họa việc sử dụng `sem_unlink()`.

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_unlink.c
#include <semaphore.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 if (argc != 2 || strcmp(argv[1], "--help") == 0)
 usageErr("%s sem-name\n", argv[0]);
 if (sem_unlink(argv[1]) == -1)
 errExit("sem_unlink");
   exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_unlink.c
```

## **53.3 Các Thao Tác Semaphore**

Giống như System V semaphore, một POSIX semaphore là một số nguyên mà hệ thống không bao giờ cho phép giảm xuống dưới 0. Tuy nhiên, các thao tác POSIX semaphore khác với các thao tác System V tương ứng ở những điểm sau:

- Các hàm để thay đổi giá trị của một semaphore — `sem_post()` và `sem_wait()` — chỉ thao tác trên một semaphore tại một thời điểm. Ngược lại, system call `semop()` của System V có thể thao tác trên nhiều semaphore trong một tập hợp.
- Các hàm `sem_post()` và `sem_wait()` tăng và giảm giá trị của một semaphore đúng một đơn vị. Ngược lại, `semop()` có thể cộng và trừ các giá trị tùy ý.
- Không có thao tác tương đương với thao tác wait-for-zero được cung cấp bởi System V semaphores (một lời gọi `semop()` trong đó trường `sops.sem_op` được chỉ định là 0).

Từ danh sách này, có vẻ như POSIX semaphores kém mạnh mẽ hơn System V semaphores. Tuy nhiên, điều này không đúng — bất cứ điều gì chúng ta có thể làm với System V semaphores cũng có thể được thực hiện với POSIX semaphores. Trong một số trường hợp, có thể cần thêm một chút nỗ lực lập trình, nhưng đối với các kịch bản điển hình, sử dụng POSIX semaphores thực sự đòi hỏi ít nỗ lập trình hơn. (API System V semaphore phức tạp hơn nhiều so với yêu cầu cho hầu hết các ứng dụng.)

## **53.3.1 Chờ trên một Semaphore**

Hàm `sem_wait()` giảm (giảm đi 1) giá trị của semaphore được tham chiếu bởi `sem`.

```
#include <semaphore.h>
int sem_wait(sem_t *sem);
                                             Returns 0 on success, or –1 on error
```

Nếu semaphore hiện có giá trị lớn hơn 0, `sem_wait()` trả về ngay lập tức. Nếu giá trị của semaphore hiện tại là 0, `sem_wait()` block cho đến khi giá trị semaphore tăng lên trên 0; vào lúc đó, semaphore sau đó được giảm đi và `sem_wait()` trả về.

Nếu một lời gọi `sem_wait()` đang bị block bị ngắt bởi một signal handler, thì nó thất bại với lỗi `EINTR`, bất kể flag `SA_RESTART` có được sử dụng khi thiết lập signal handler với `sigaction()` hay không. (Trên một số hiện thực UNIX khác, `SA_RESTART` thực sự khiến `sem_wait()` tự động khởi động lại.)

Chương trình trong Listing 53-3 cung cấp một giao diện dòng lệnh cho hàm `sem_wait()`. Chúng ta sẽ sớm minh họa việc sử dụng chương trình này.

**Listing 53-3:** Sử dụng `sem_wait()` để giảm một POSIX semaphore

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_wait.c
#include <semaphore.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 sem_t *sem;
 if (argc < 2 || strcmp(argv[1], "--help") == 0)
 usageErr("%s sem-name\n", argv[0]);
 sem = sem_open(argv[1], 0);
 if (sem == SEM_FAILED)
 errExit("sem_open");
 if (sem_wait(sem) == -1)
 errExit("sem_wait");
 printf("%ld sem_wait() succeeded\n", (long) getpid());
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_wait.c
```

Hàm `sem_trywait()` là phiên bản không blocking của `sem_wait()`.

```
#include <semaphore.h>
int sem_trywait(sem_t *sem);
                                             Returns 0 on success, or –1 on error
```

Nếu thao tác giảm không thể được thực hiện ngay lập tức, `sem_trywait()` thất bại với lỗi `EAGAIN`.

Hàm `sem_timedwait()` là một biến thể khác của `sem_wait()`. Nó cho phép người gọi chỉ định giới hạn thời gian mà lời gọi sẽ block.

```
#define _XOPEN_SOURCE 600
#include <semaphore.h>
int sem_timedwait(sem_t *sem, const struct timespec *abs_timeout);
                                             Returns 0 on success, or –1 on error
```

Nếu một lời gọi `sem_timedwait()` hết thời gian chờ mà không thể giảm semaphore, thì lời gọi thất bại với lỗi `ETIMEDOUT`.

Đối số `abs_timeout` là một cấu trúc `timespec` (Mục 23.4.2) chỉ định thời gian chờ tối đa dưới dạng giá trị tuyệt đối tính bằng giây và nanosecond kể từ Epoch. Nếu chúng ta muốn thực hiện timeout tương đối, thì chúng ta phải lấy giá trị hiện tại của clock `CLOCK_REALTIME` bằng cách sử dụng `clock_gettime()` và cộng thêm lượng thời gian cần thiết vào giá trị đó để tạo ra một cấu trúc `timespec` phù hợp để sử dụng với `sem_timedwait()`.

Hàm `sem_timedwait()` ban đầu được quy định trong POSIX.1d (1999) và không có sẵn trên tất cả các hiện thực UNIX.

## **53.3.2 Post một Semaphore**

Hàm `sem_post()` tăng (tăng lên 1) giá trị của semaphore được tham chiếu bởi `sem`.

```
#include <semaphore.h>
int sem_post(sem_t *sem);
                                             Returns 0 on success, or –1 on error
```

Nếu giá trị của semaphore là 0 trước khi gọi `sem_post()`, và một số process (hoặc thread) khác đang bị block chờ để giảm semaphore, thì process đó được đánh thức và lời gọi `sem_wait()` của nó tiến hành giảm semaphore. Nếu nhiều process (hoặc thread) đang bị block trong `sem_wait()`, thì, nếu các process đang được lập lịch theo chính sách round-robin time-sharing mặc định, không xác định được process nào sẽ được đánh thức và được phép giảm semaphore. (Giống như các thao tác System V tương ứng của chúng, POSIX semaphores chỉ là cơ chế đồng bộ hóa, không phải là cơ chế xếp hàng.)

> SUSv3 quy định rằng nếu các process hoặc thread đang được thực thi theo chính sách lập lịch realtime, thì process hoặc thread sẽ được đánh thức là process hoặc thread có độ ưu tiên cao nhất đã chờ lâu nhất.

Như với System V semaphores, việc tăng một POSIX semaphore tương ứng với việc giải phóng một số tài nguyên dùng chung để process hoặc thread khác sử dụng.

Chương trình trong Listing 53-4 cung cấp một giao diện dòng lệnh cho hàm `sem_post()`. Chúng ta sẽ sớm minh họa việc sử dụng chương trình này.

```
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_post.c
#include <semaphore.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 sem_t *sem;
 if (argc != 2)
 usageErr("%s sem-name\n", argv[0]);
 sem = sem_open(argv[1], 0);
 if (sem == SEM_FAILED)
 errExit("sem_open");
 if (sem_post(sem) == -1)
 errExit("sem_post");
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_post.c
```

## **53.3.3 Lấy Giá Trị Hiện Tại của một Semaphore**

Hàm `sem_getvalue()` trả về giá trị hiện tại của semaphore được tham chiếu bởi `sem` trong biến `int` được trỏ tới bởi `sval`.

```
#include <semaphore.h>
int sem_getvalue(sem_t *sem, int *sval);
                                             Returns 0 on success, or –1 on error
```

Nếu một hoặc nhiều process (hoặc thread) hiện đang bị block chờ để giảm giá trị của semaphore, thì giá trị được trả về trong `sval` phụ thuộc vào hiện thực. SUSv3 cho phép hai khả năng: 0 hoặc một số âm có giá trị tuyệt đối là số lượng người chờ đang bị block trong `sem_wait()`. Linux và một số hiện thực khác áp dụng hành vi trước; một vài hiện thực khác áp dụng hành vi sau.

> Mặc dù việc trả về `sval` âm nếu có người chờ bị block có thể hữu ích, đặc biệt cho mục đích gỡ lỗi, SUSv3 không yêu cầu hành vi này vì các kỹ thuật mà một số hệ thống sử dụng để hiện thực POSIX semaphores một cách hiệu quả không (thực tế, không thể) ghi lại số lượng người chờ bị block.

Lưu ý rằng vào thời điểm `sem_getvalue()` trả về, giá trị được trả về trong `sval` có thể đã lỗi thời. Một chương trình phụ thuộc vào thông tin được trả về bởi `sem_getvalue()` không thay đổi vào thời điểm của một thao tác tiếp theo sẽ bị ảnh hưởng bởi điều kiện race time-of-check, time-of-use (Mục 38.6).

Chương trình trong Listing 53-5 sử dụng `sem_getvalue()` để lấy giá trị của semaphore được đặt tên trong đối số dòng lệnh của nó, sau đó hiển thị giá trị đó trên standard output.

**Listing 53-5:** Sử dụng `sem_getvalue()` để lấy giá trị của một POSIX semaphore

```
–––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_getvalue.c
#include <semaphore.h>
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
 int value;
 sem_t *sem;
 if (argc != 2)
 usageErr("%s sem-name\n", argv[0]);
 sem = sem_open(argv[1], 0);
 if (sem == SEM_FAILED)
 errExit("sem_open");
 if (sem_getvalue(sem, &value) == -1)
 errExit("sem_getvalue");
 printf("%d\n", value);
 exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––– psem/psem_getvalue.c
```

#### **Ví dụ**

Phiên log shell sau đây minh họa việc sử dụng các chương trình chúng ta đã trình bày cho đến nay trong chương này. Chúng ta bắt đầu bằng cách tạo một semaphore có giá trị ban đầu là không, sau đó khởi động một chương trình ở nền cố gắng giảm semaphore:

```
$ ./psem_create -c /demo 600 0
$ ./psem_wait /demo &
[1] 31208
```

Lệnh nền bị block, vì giá trị semaphore hiện tại là 0 và do đó không thể giảm thêm.

Sau đó chúng ta lấy giá trị semaphore:

```
$ ./psem_getvalue /demo
0
```

Chúng ta thấy giá trị 0 ở trên. Trên một số hiện thực khác, chúng ta có thể thấy giá trị –1, cho biết rằng một process đang chờ trên semaphore.

Sau đó chúng ta thực thi một lệnh tăng semaphore. Điều này khiến `sem_wait()` bị block trong chương trình nền hoàn thành:

```
$ ./psem_post /demo
$ 31208 sem_wait() succeeded
```

(Dòng cuối cùng của đầu ra ở trên cho thấy dấu nhắc shell trộn với đầu ra của job nền.)

Chúng ta nhấn Enter để xem dấu nhắc shell tiếp theo, điều này cũng khiến shell báo cáo về job nền đã kết thúc, sau đó thực hiện các thao tác tiếp theo trên semaphore:

```
Press Enter
[1]- Done ./psem_wait /demo
$ ./psem_post /demo Increment semaphore
$ ./psem_getvalue /demo Retrieve semaphore value
1
$ ./psem_unlink /demo We're done with this semaphore
```

## **53.4 Unnamed Semaphores**

Unnamed semaphores (còn được gọi là memory-based semaphores) là các biến kiểu `sem_t` được lưu trữ trong bộ nhớ được cấp phát bởi ứng dụng. Semaphore được cung cấp cho các process hoặc thread sử dụng nó bằng cách đặt nó vào một vùng bộ nhớ mà chúng chia sẻ.

Các thao tác trên unnamed semaphores sử dụng các hàm tương tự (`sem_wait()`, `sem_post()`, `sem_getvalue()`, v.v.) được sử dụng để thao tác trên named semaphores. Ngoài ra, cần thêm hai hàm nữa:

- Hàm `sem_init()` khởi tạo một semaphore và thông báo cho hệ thống biết semaphore sẽ được chia sẻ giữa các process hay giữa các thread của một process duy nhất.
- Hàm `sem_destroy(sem)` hủy một semaphore.

Các hàm này không nên được sử dụng với named semaphores.

#### **Unnamed versus named semaphores**

Sử dụng unnamed semaphore cho phép chúng ta tránh phải tạo tên cho một semaphore. Điều này có thể hữu ích trong các trường hợp sau:

- Một semaphore được chia sẻ giữa các thread không cần tên. Biến một unnamed semaphore thành biến shared (global hoặc heap) tự động làm cho nó có thể truy cập được bởi tất cả các thread.
- Một semaphore được chia sẻ giữa các process liên quan không cần tên. Nếu một process cha cấp phát một unnamed semaphore trong một vùng shared memory (ví dụ: một shared anonymous mapping), thì một process con tự động kế thừa mapping và do đó semaphore như một phần của thao tác `fork()`.

 Nếu chúng ta đang xây dựng một cấu trúc dữ liệu động (ví dụ: một cây nhị phân), mỗi phần tử của nó yêu cầu một semaphore liên kết, thì cách tiếp cận đơn giản nhất là cấp phát một unnamed semaphore trong mỗi phần tử. Mở một named semaphore cho mỗi phần tử sẽ yêu cầu chúng ta thiết kế một quy ước để tạo tên semaphore (duy nhất) cho mỗi phần tử và quản lý các tên đó (ví dụ: unlink chúng khi chúng không còn cần thiết nữa).

## **53.4.1 Khởi tạo một Unnamed Semaphore**

Hàm `sem_init()` khởi tạo unnamed semaphore được trỏ tới bởi `sem` với giá trị được chỉ định bởi `value`.

```
#include <semaphore.h>
int sem_init(sem_t *sem, int pshared, unsigned int value);
                                            Returns 0 on success, or –1 on error
```

Đối số `pshared` cho biết semaphore sẽ được chia sẻ giữa các thread hay giữa các process.

- Nếu `pshared` là 0, thì semaphore được chia sẻ giữa các thread của process đang gọi. Trong trường hợp này, `sem` thường được chỉ định là địa chỉ của một biến global hoặc một biến được cấp phát trên heap. Một thread-shared semaphore có process persistence; nó bị hủy khi process kết thúc.
- Nếu `pshared` khác không, thì semaphore được chia sẻ giữa các process. Trong trường hợp này, `sem` phải là địa chỉ của một vị trí trong một vùng shared memory (một POSIX shared memory object, một shared mapping được tạo bằng cách sử dụng `mmap()`, hoặc một System V shared memory segment). Semaphore tồn tại chừng nào shared memory mà nó nằm trong đó còn tồn tại. (Các vùng shared memory được tạo bởi hầu hết các kỹ thuật này có kernel persistence. Ngoại lệ là shared anonymous mappings, chỉ tồn tại chừng nào có ít nhất một process duy trì mapping.) Vì một process con được tạo thông qua `fork()` kế thừa các memory mapping của process cha, các process-shared semaphores được kế thừa bởi process con của một `fork()`, và process cha và process con có thể sử dụng các semaphore này để đồng bộ hóa các hành động của chúng.

Đối số `pshared` là cần thiết vì những lý do sau:

- Một số hiện thực không hỗ trợ process-shared semaphores. Trên các hệ thống này, việc chỉ định giá trị khác không cho `pshared` khiến `sem_init()` trả về lỗi. Linux không hỗ trợ unnamed process-shared semaphores cho đến kernel 2.6 và sự ra đời của hiện thực threading NPTL. (Hiện thực LinuxThreads cũ hơn của `sem_init()` thất bại với lỗi `ENOSYS` nếu giá trị khác không được chỉ định cho `pshared`.)
- Trên các hiện thực hỗ trợ cả process-shared và thread-shared semaphores, việc chỉ định loại chia sẻ nào được yêu cầu có thể cần thiết vì hệ thống phải thực hiện các hành động đặc biệt để hỗ trợ chia sẻ được yêu cầu. Cung cấp thông tin này cũng có thể cho phép hệ thống thực hiện các tối ưu hóa tùy thuộc vào loại chia sẻ.

Hiện thực NPTL `sem_init()` bỏ qua `pshared`, vì không cần hành động đặc biệt nào cho bất kỳ loại chia sẻ nào. Tuy nhiên, các ứng dụng portable và future-proof nên chỉ định giá trị phù hợp cho `pshared`.

> Đặc tả SUSv3 cho `sem_init()` định nghĩa giá trị trả về khi thất bại là –1, nhưng không đưa ra bất kỳ tuyên bố nào về giá trị trả về khi thành công. Tuy nhiên, các trang hướng dẫn trên hầu hết các hiện thực UNIX hiện đại đều ghi lại giá trị trả về 0 khi thành công. (Một ngoại lệ đáng chú ý là Solaris, nơi mô tả về giá trị trả về tương tự như đặc tả SUSv3. Tuy nhiên, kiểm tra mã nguồn OpenSolaris cho thấy, trên hiện thực đó, `sem_init()` thực sự trả về 0 khi thành công.) SUSv4 khắc phục tình trạng này, quy định rằng `sem_init()` sẽ trả về 0 khi thành công.

Không có cài đặt quyền nào được liên kết với một unnamed semaphore (tức là `sem_init()` không có tương tự với đối số `mode` của `sem_open()`). Quyền truy cập vào một unnamed semaphore được điều chỉnh bởi các quyền được cấp cho process đối với vùng shared memory cơ sở.

SUSv3 quy định rằng việc khởi tạo một unnamed semaphore đã được khởi tạo dẫn đến hành vi không xác định. Nói cách khác, chúng ta phải thiết kế ứng dụng của mình sao cho chỉ một process hoặc thread gọi `sem_init()` để khởi tạo một semaphore.

Như với named semaphores, SUSv3 nói rằng kết quả là không xác định nếu chúng ta cố thực hiện các thao tác trên một bản sao của biến `sem_t` có địa chỉ được truyền dưới dạng đối số `sem` của `sem_init()`. Các thao tác phải luôn được thực hiện chỉ trên semaphore "gốc".

#### **Chương trình ví dụ**

Trong Mục 30.1.2, chúng ta đã trình bày một chương trình (Listing 30-2) sử dụng mutex để bảo vệ một critical section trong đó hai thread truy cập cùng một biến global. Chương trình trong Listing 53-6 giải quyết cùng một vấn đề bằng cách sử dụng một unnamed thread-shared semaphore.

**Listing 53-6:** Sử dụng một POSIX unnamed semaphore để bảo vệ quyền truy cập vào một biến global

```
––––––––––––––––––––––––––––––––––––––––––––––––––– psem/thread_incr_psem.c
#include <semaphore.h>
#include <pthread.h>
#include "tlpi_hdr.h"
static int glob = 0;
static sem_t sem;
static void *  /* Loop 'arg' times incrementing 'glob' */
threadFunc(void *arg)
{
 int loops = *((int *) arg);
 int loc, j;
 for (j = 0; j < loops; j++) {
 if (sem_wait(&sem) == -1)
 errExit("sem_wait");
 loc = glob;
 loc++;
 glob = loc;
 if (sem_post(&sem) == -1)
 errExit("sem_post");
 }
 return NULL;
}
int
main(int argc, char *argv[])
{
 pthread_t t1, t2;
 int loops, s;
 loops = (argc > 1) ? getInt(argv[1], GN_GT_0, "num-loops") : 10000000;
 /* Initialize a thread-shared mutex with the value 1 */
 if (sem_init(&sem, 0, 1) == -1)
 errExit("sem_init");
   /* Create two threads that increment 'glob' */
 s = pthread_create(&t1, NULL, threadFunc, &loops);
 if (s != 0)
 errExitEN(s, "pthread_create");
 s = pthread_create(&t2, NULL, threadFunc, &loops);
 if (s != 0)
 errExitEN(s, "pthread_create");
   /* Wait for threads to terminate */
 s = pthread_join(t1, NULL);
 if (s != 0)
 errExitEN(s, "pthread_join");
 s = pthread_join(t2, NULL);
 if (s != 0)
 errExitEN(s, "pthread_join");
 printf("glob = %d\n", glob);
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––– psem/thread_incr_psem.c
```

## **53.4.2 Hủy một Unnamed Semaphore**

Hàm `sem_destroy()` hủy semaphore `sem`, phải là một unnamed semaphore đã được khởi tạo trước đó bằng cách sử dụng `sem_init()`. Việc hủy một semaphore chỉ an toàn nếu không có process hoặc thread nào đang chờ trên nó.

```
#include <semaphore.h>
int sem_destroy(sem_t *sem);
                                             Returns 0 on success, or –1 on error
```

Sau khi một unnamed semaphore đã bị hủy với `sem_destroy()`, nó có thể được khởi tạo lại với `sem_init()`.

Một unnamed semaphore nên bị hủy trước khi bộ nhớ cơ sở của nó được giải phóng. Ví dụ, nếu semaphore là một biến được tự động cấp phát, nó nên được hủy trước khi hàm chứa nó trả về. Nếu semaphore nằm trong một vùng POSIX shared memory, thì nó nên bị hủy sau khi tất cả các process đã ngừng sử dụng semaphore và trước khi shared memory object được unlink với `shm_unlink()`.

Trên một số hiện thực, việc bỏ qua các lời gọi `sem_destroy()` không gây ra vấn đề gì. Tuy nhiên, trên các hiện thực khác, việc không gọi `sem_destroy()` có thể dẫn đến rò rỉ tài nguyên. Các ứng dụng portable nên gọi `sem_destroy()` để tránh những vấn đề như vậy.

## **53.5 So Sánh với Các Kỹ Thuật Đồng Bộ Hóa Khác**

Trong phần này, chúng ta so sánh POSIX semaphores với hai kỹ thuật đồng bộ hóa khác: System V semaphores và mutex.

#### **POSIX semaphores versus System V semaphores**

Cả POSIX semaphores và System V semaphores đều có thể được sử dụng để đồng bộ hóa các hành động của các process. Mục 51.2 đã liệt kê nhiều lợi thế của POSIX IPC so với System V IPC: giao diện POSIX IPC đơn giản hơn và nhất quán hơn với mô hình file UNIX truyền thống, và các POSIX IPC object được tính tham chiếu, đơn giản hóa việc xác định khi nào cần xóa một IPC object. Những lợi thế chung này cũng áp dụng cho trường hợp cụ thể của POSIX (named) semaphores so với System V semaphores.

POSIX semaphores có những lợi thế tiếp theo sau đây so với System V semaphores:

- Giao diện POSIX semaphore đơn giản hơn nhiều so với giao diện System V semaphore. Sự đơn giản này đạt được mà không mất đi sức mạnh chức năng.
- POSIX named semaphores loại bỏ vấn đề khởi tạo liên quan đến System V semaphores (Mục 47.5).
- Dễ dàng hơn khi liên kết một POSIX unnamed semaphore với một đối tượng bộ nhớ được cấp phát động: semaphore có thể đơn giản được nhúng vào bên trong đối tượng.
- Trong các kịch bản có mức độ tranh chấp cao cho một semaphore (tức là, các thao tác trên semaphore thường xuyên bị block vì một process khác đã đặt semaphore thành giá trị ngăn thao tác tiến hành ngay lập tức), thì hiệu suất của POSIX semaphores và System V semaphores là tương tự nhau. Tuy nhiên, trong các trường hợp có mức độ tranh chấp thấp cho một semaphore (tức là, giá trị của semaphore sao cho các thao tác thường có thể tiến hành mà không bị block), thì POSIX semaphores thực hiện tốt hơn đáng kể so với System V semaphores. (Trên các hệ thống được tác giả kiểm thử, sự khác biệt về hiệu suất vượt quá một bậc độ lớn; xem Bài tập 53-4.) POSIX semaphores thực hiện tốt hơn nhiều trong trường hợp này vì cách chúng được hiện thực chỉ yêu cầu một system call khi xảy ra tranh chấp, trong khi các thao tác System V semaphore luôn yêu cầu một system call, bất kể tranh chấp.

Tuy nhiên, POSIX semaphores cũng có những nhược điểm sau so với System V semaphores:

- POSIX semaphores ít portable hơn một chút. (Trên Linux, named semaphores chỉ được hỗ trợ kể từ kernel 2.6.)
- POSIX semaphores không cung cấp tương đương với tính năng undo của System V semaphore. (Tuy nhiên, như chúng ta đã lưu ý trong Mục 47.8, tính năng này có thể không hữu ích trong một số trường hợp nhất định.)

#### **POSIX semaphores versus Pthreads mutexes**

Cả POSIX semaphores và Pthreads mutexes đều có thể được sử dụng để đồng bộ hóa các hành động của các thread trong cùng một process, và hiệu suất của chúng là tương tự nhau. Tuy nhiên, mutex thường được ưa thích hơn, vì tính chất sở hữu của mutex thực thi cấu trúc code tốt (chỉ thread khóa một mutex mới có thể mở khóa nó). Ngược lại, một thread có thể tăng một semaphore đã bị giảm bởi một thread khác. Tính linh hoạt này có thể dẫn đến các thiết kế đồng bộ hóa có cấu trúc kém. (Vì lý do này, semaphores đôi khi được gọi là "gotos" của lập trình đồng thời.)

Có một trường hợp mà mutex không thể được sử dụng trong một ứng dụng đa luồng và semaphores có thể do đó được ưa thích hơn. Vì nó là async-signal-safe (xem Bảng 21-1, trang 426), hàm `sem_post()` có thể được sử dụng từ bên trong một signal handler để đồng bộ hóa với một thread khác. Điều này không thể thực hiện được với mutex, vì các hàm Pthreads để thao tác trên mutex không phải là async-signal-safe. Tuy nhiên, vì thường tốt hơn khi xử lý các asynchronous signal bằng cách chấp nhận chúng bằng `sigwaitinfo()` (hoặc tương tự), thay vì sử dụng signal handler (xem Mục 33.2.4), lợi thế này của semaphores so với mutex hiếm khi được yêu cầu.

## **53.6 Giới Hạn Semaphore**

SUSv3 định nghĩa hai giới hạn áp dụng cho semaphores:

`SEM_NSEMS_MAX`

Đây là số lượng tối đa các POSIX semaphores mà một process có thể có. SUSv3 yêu cầu giới hạn này ít nhất là 256. Trên Linux, số lượng POSIX semaphores chỉ bị giới hạn bởi bộ nhớ khả dụng.

`SEM_VALUE_MAX`

Đây là giá trị tối đa mà một POSIX semaphore có thể đạt tới. Semaphores có thể nhận bất kỳ giá trị nào từ 0 đến giới hạn này. SUSv3 yêu cầu giới hạn này ít nhất là 32.767; hiện thực Linux cho phép các giá trị lên đến `INT_MAX` (2.147.483.647 trên Linux/x86-32).

## **53.7 Tóm Tắt**

POSIX semaphores cho phép các process hoặc thread đồng bộ hóa các hành động của chúng. POSIX semaphores có hai loại: named và unnamed. Một named semaphore được xác định bởi một tên và có thể được chia sẻ bởi bất kỳ process nào có quyền mở semaphore. Một unnamed semaphore không có tên, nhưng các process hoặc thread có thể chia sẻ cùng một semaphore bằng cách đặt nó vào một vùng bộ nhớ mà chúng chia sẻ (ví dụ: trong một POSIX shared memory object cho việc chia sẻ giữa các process, hoặc trong một biến global cho việc chia sẻ giữa các thread).

Giao diện POSIX semaphore đơn giản hơn so với giao diện System V semaphore. Các semaphore được cấp phát và thao tác riêng lẻ, và các thao tác wait và post điều chỉnh giá trị của một semaphore đi một đơn vị.

POSIX semaphores có một số lợi thế so với System V semaphores, nhưng chúng ít portable hơn một chút. Để đồng bộ hóa trong các ứng dụng đa luồng, mutex thường được ưa thích hơn semaphores.

#### **Tài liệu tham khảo thêm**

[Stevens, 1999] cung cấp một trình bày thay thế về POSIX semaphores và trình bày các hiện thực user-space sử dụng nhiều cơ chế IPC khác (FIFO, memory-mapped file, và System V semaphores). [Butenhof, 1996] mô tả việc sử dụng POSIX semaphores trong các ứng dụng đa luồng.

## **53.8 Bài Tập**

- **53-1.** Viết lại các chương trình trong Listing 48-2 và Listing 48-3 (Mục 48.4) dưới dạng một ứng dụng có thread, với hai thread truyền dữ liệu cho nhau thông qua một buffer global và sử dụng POSIX semaphores để đồng bộ hóa.
- **53-2.** Sửa đổi chương trình trong Listing 53-3 (psem\_wait.c) để sử dụng `sem_timedwait()` thay vì `sem_wait()`. Chương trình nên nhận thêm một đối số dòng lệnh chỉ định số giây (tương đối) được sử dụng làm timeout cho lời gọi `sem_timedwait()`.
- **53-3.** Thiết kế một hiện thực của POSIX semaphores sử dụng System V semaphores.
- **53-4.** Trong Mục 53.5, chúng ta đã lưu ý rằng POSIX semaphores thực hiện tốt hơn nhiều so với System V semaphores trong trường hợp semaphore không bị tranh chấp. Viết hai chương trình (một cho mỗi loại semaphore) để xác minh điều này. Mỗi chương trình chỉ cần tăng và giảm một semaphore một số lần được chỉ định. So sánh thời gian cần thiết cho hai chương trình.
