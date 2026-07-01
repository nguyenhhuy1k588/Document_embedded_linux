## Chương 29
# **THREADS: GIỚI THIỆU**

Trong chương này và một số chương tiếp theo, chúng ta sẽ mô tả POSIX threads, thường được gọi là Pthreads. Chúng ta sẽ không cố gắng bao quát toàn bộ API Pthreads, vì nó khá lớn. Các nguồn thông tin thêm về threads được liệt kê ở cuối chương này.

Các chương này chủ yếu mô tả hành vi chuẩn được chỉ định cho API Pthreads. Trong Mục [33.5](#page-72-0), chúng ta thảo luận về những điểm mà hai triển khai threading chính trên Linux—LinuxThreads và Native POSIX Threads Library (NPTL)—khác với tiêu chuẩn.

Trong chương này, chúng ta cung cấp tổng quan về hoạt động của threads, sau đó xem xét cách các threads được tạo ra và cách chúng kết thúc. Chúng ta kết thúc bằng một cuộc thảo luận về một số yếu tố có thể ảnh hưởng đến việc lựa chọn phương pháp đa luồng so với phương pháp đa tiến trình khi thiết kế ứng dụng.

# **29.1 Tổng quan**

Giống như process, thread là một cơ chế cho phép ứng dụng thực hiện nhiều tác vụ đồng thời. Một process duy nhất có thể chứa nhiều thread, như minh họa trong [Hình 29-1](#page-1-0). Tất cả các thread này đều độc lập thực thi cùng một chương trình, và chúng đều chia sẻ cùng một bộ nhớ global, bao gồm các đoạn dữ liệu đã khởi tạo, dữ liệu chưa khởi tạo, và đoạn heap. (Một process UNIX truyền thống chỉ là một trường hợp đặc biệt của một process đa luồng; đó là một process chỉ chứa một thread duy nhất.)

Chúng ta đã đơn giản hóa đôi chút trong [Hình 29-1.](#page-1-0) Đặc biệt, vị trí của các stack per-thread có thể xen kẽ với shared libraries và các vùng shared memory, tùy thuộc vào thứ tự tạo thread, tải shared libraries và gắn kết các vùng shared memory. Hơn nữa, vị trí của các stack per-thread có thể thay đổi tùy thuộc vào bản phân phối Linux.

Các thread trong một process có thể thực thi đồng thời. Trên hệ thống đa bộ xử lý, nhiều thread có thể thực thi song song. Nếu một thread bị chặn trên I/O, các thread khác vẫn đủ điều kiện để thực thi. (Mặc dù đôi khi hữu ích khi tạo một thread riêng biệt chỉ với mục đích thực hiện I/O, nhưng thường ưu tiên sử dụng một trong các mô hình I/O thay thế mà chúng ta mô tả trong Chương 63.)

```text
Virtual memory address
    (hexadecimal)
                        ┌─────────────────────────────┐
    0xC0000000          │      argc, environ          │
                        ├─────────────────────────────┤
                        │  Stack for main thread      │
                        ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ --┤
                        │            │                │
                        │            ▼                │
                        │                             │
                        │                             │
                        ├─────────────────────────────┤
                        │   Stack for thread 3        │
                        ├─────────────────────────────┤
                        │   Stack for thread 2        │
                        ├─────────────────────────────┤
                        │   Stack for thread 1        │
                        ├─────────────────────────────┤
    0x40000000          │   Shared libraries,         │
TASK_UNMAPPED_BASE      │   shared memory             │
                        │                             │
                        │                             │
                        │            ▲                │
                        ├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ --┤
                        │          Heap               │
                        ├─────────────────────────────┤
                        │ Uninitialized data (bss)    │
                        ├─────────────────────────────┤
                        │    Initialized data         │
        ▲               │                             │◄── thread 3 executing here
        │               ├─────────────────────────────┤
        │               │                             │◄── main thread executing here
        │               │   Text (program code)       │
increasing virtual      │                             │◄── thread 1 executing here
addresses               │                             │
        │               ├─────────────────────────────┤◄── thread 2 executing here
        │               │                             │
                        │                             │
    0x08048000          ├─────────────────────────────┤
                        │                             │
    0x00000000          └─────────────────────────────┘
```

<span id="page-1-1"></span><span id="page-1-0"></span>**Hình 29-1:** Bốn thread đang thực thi trong một process (Linux/x86-32)

Threads mang lại lợi thế so với process trong một số ứng dụng. Hãy xem xét cách tiếp cận UNIX truyền thống để đạt được tính đồng thời bằng cách tạo nhiều process. Một ví dụ về điều này là thiết kế server mạng trong đó một process cha chấp nhận các kết nối đến từ client, sau đó sử dụng `fork()` để tạo một process con riêng biệt để xử lý giao tiếp với từng client (tham khảo Mục 60.3). Thiết kế như vậy cho phép phục vụ nhiều client đồng thời. Mặc dù cách tiếp cận này hoạt động tốt trong nhiều tình huống, nhưng nó có những hạn chế sau trong một số ứng dụng:

-  Chia sẻ thông tin giữa các process là khó khăn. Vì process cha và process con không chia sẻ bộ nhớ (ngoại trừ đoạn text chỉ đọc), chúng ta phải sử dụng một số hình thức interprocess communication để trao đổi thông tin giữa các process.
-  Tạo process bằng `fork()` tương đối tốn kém. Ngay cả với kỹ thuật copy-on-write được mô tả trong Mục 24.2.2, sự cần thiết phải sao chép các thuộc tính process khác nhau như page table và file descriptor table có nghĩa là một lệnh gọi `fork()` vẫn tốn thời gian.

#### Threads giải quyết cả hai vấn đề này:

-  Chia sẻ thông tin giữa các thread là dễ dàng và nhanh chóng. Chỉ cần sao chép dữ liệu vào các biến chia sẻ (global hoặc heap). Tuy nhiên, để tránh các vấn đề có thể xảy ra khi nhiều thread cố gắng cập nhật cùng một thông tin, chúng ta phải sử dụng các kỹ thuật synchronization được mô tả trong Chương [30.](#page-14-0)
-  Tạo thread nhanh hơn tạo process—thường nhanh hơn mười lần hoặc hơn. (Trên Linux, thread được triển khai bằng system call `clone()`, và Bảng 28-3, trang 610, cho thấy sự khác biệt về tốc độ giữa `fork()` và `clone()`.) Tạo thread nhanh hơn vì nhiều thuộc tính phải được sao chép trong process con được tạo bởi `fork()` thay vào đó được chia sẻ giữa các thread. Đặc biệt, không cần sao chép copy-on-write các page bộ nhớ, cũng không cần sao chép page table.

Bên cạnh bộ nhớ global, các thread cũng chia sẻ một số thuộc tính khác (tức là các thuộc tính này là global đối với một process, thay vì đặc thù cho một thread). Các thuộc tính này bao gồm:

-  process ID và process ID cha;
-  process group ID và session ID;
-  controlling terminal;
-  thông tin xác thực của process (user và group ID);
-  các file descriptor đang mở;
-  record lock được tạo bằng `fcntl()`;
-  signal disposition;
-  thông tin liên quan đến file system: umask, thư mục làm việc hiện tại, và thư mục gốc;
-  interval timer (`setitimer()`) và POSIX timer (`timer_create()`);
-  giá trị System V semaphore undo (semadj) (Mục 47.8);
-  giới hạn tài nguyên;
-  thời gian CPU đã tiêu thụ (như trả về bởi `times()`);
-  tài nguyên đã tiêu thụ (như trả về bởi `getrusage()`); và
-  nice value (được đặt bởi `setpriority()` và `nice()`).

Trong số các thuộc tính riêng biệt cho mỗi thread là:

-  thread ID (Mục [29.5\)](#page-7-0);
-  signal mask;
-  thread-specific data (Mục [31.3](#page-42-0));
-  alternate signal stack (`sigaltstack()`);
-  biến `errno`;
-  môi trường dấu phẩy động (xem `fenv(3)`);
-  chính sách và độ ưu tiên realtime scheduling (Mục 35.2 và 35.3);
-  CPU affinity (đặc thù Linux, được mô tả trong Mục 35.4);
-  capabilities (đặc thù Linux, được mô tả trong Chương 39); và
-  stack (biến cục bộ và thông tin liên kết lệnh gọi hàm).

Như có thể thấy từ [Hình 29-1,](#page-1-0) tất cả các stack per-thread đều nằm trong cùng một virtual address space. Điều này có nghĩa là, với một pointer phù hợp, các thread có thể chia sẻ dữ liệu trên stack của nhau. Điều này đôi khi hữu ích, nhưng đòi hỏi lập trình cẩn thận để xử lý sự phụ thuộc xuất phát từ thực tế là một biến cục bộ chỉ còn hợp lệ trong thời gian tồn tại của stack frame mà nó nằm trong đó. (Nếu một hàm trả về, vùng bộ nhớ được sử dụng bởi stack frame của nó có thể được tái sử dụng bởi một lệnh gọi hàm sau đó. Nếu thread kết thúc, một thread mới có thể tái sử dụng vùng bộ nhớ được sử dụng cho stack của thread đã kết thúc.) Không xử lý đúng sự phụ thuộc này có thể tạo ra các bug khó theo dõi.

# **29.2 Chi tiết nền tảng của API Pthreads**

Vào cuối những năm 1980 và đầu những năm 1990, một số API threading khác nhau đã tồn tại. Năm 1995, POSIX.1c đã chuẩn hóa API POSIX threads, và tiêu chuẩn này sau đó được đưa vào SUSv3.

Một số khái niệm áp dụng cho toàn bộ API Pthreads, và chúng ta giới thiệu ngắn gọn những điều này trước khi xem xét chi tiết API.

## **Kiểu dữ liệu Pthreads**

API Pthreads định nghĩa một số kiểu dữ liệu, một số trong đó được liệt kê trong Bảng 29-1. Chúng ta mô tả hầu hết các kiểu dữ liệu này trong các trang tiếp theo.

**Bảng 29-1:** Kiểu dữ liệu Pthreads

| Kiểu dữ liệu        | Mô tả                                   |
|---------------------|-----------------------------------------|
| `pthread_t`           | Định danh thread                        |
| `pthread_mutex_t`     | Mutex                                   |
| `pthread_mutexattr_t` | Đối tượng thuộc tính mutex              |
| `pthread_cond_t`      | Condition variable                      |
| `pthread_condattr_t`  | Đối tượng thuộc tính condition variable |
| `pthread_key_t`       | Key cho thread-specific data            |
| `pthread_once_t`      | Context điều khiển khởi tạo một lần    |
| `pthread_attr_t`      | Đối tượng thuộc tính thread             |

SUSv3 không chỉ định cách biểu diễn các kiểu dữ liệu này, và các chương trình di động nên coi chúng như dữ liệu không trong suốt (opaque data). Điều này có nghĩa là một chương trình nên tránh mọi sự phụ thuộc vào kiến thức về cấu trúc hoặc nội dung của một biến thuộc một trong các kiểu này. Đặc biệt, chúng ta không thể so sánh các biến thuộc các kiểu này bằng toán tử `==` của C.

### **Threads và errno**

Trong API UNIX truyền thống, `errno` là một biến integer global. Tuy nhiên, điều này không đủ cho các chương trình đa luồng. Nếu một thread thực hiện lệnh gọi hàm trả về lỗi trong một biến `errno` global, thì điều này sẽ làm nhầm lẫn các thread khác cũng có thể đang thực hiện lệnh gọi hàm và kiểm tra `errno`. Nói cách khác, race condition sẽ xảy ra. Do đó, trong các chương trình đa luồng, mỗi thread có giá trị `errno` riêng của nó. Trên Linux, một `errno` đặc thù theo thread được đạt được theo cách tương tự như hầu hết các triển khai UNIX khác: `errno` được định nghĩa là một macro mở rộng thành một lệnh gọi hàm trả về một lvalue có thể sửa đổi, khác biệt với mỗi thread. (Vì lvalue có thể sửa đổi, vẫn có thể viết các câu lệnh gán dạng `errno = value` trong các chương trình đa luồng.)

Tóm lại, cơ chế `errno` đã được điều chỉnh cho các thread theo cách không thay đổi việc báo cáo lỗi từ API UNIX truyền thống.

> Tiêu chuẩn POSIX.1 gốc theo cách dùng K&R C cho phép một chương trình khai báo `errno` là `extern int errno`. SUSv3 không cho phép cách dùng này (sự thay đổi thực sự xảy ra vào năm 1995 trong POSIX.1c). Ngày nay, một chương trình bắt buộc phải khai báo `errno` bằng cách include `<errno.h>`, điều này cho phép triển khai `errno` per-thread.

### **Giá trị trả về từ các hàm Pthreads**

Phương pháp truyền thống để trả về trạng thái từ system call và một số library function là trả về 0 khi thành công và –1 khi gặp lỗi, với `errno` được đặt để chỉ ra lỗi. Các hàm trong API Pthreads hoạt động khác. Tất cả các hàm Pthreads trả về 0 khi thành công hoặc một giá trị dương khi gặp lỗi. Giá trị lỗi là một trong các giá trị giống như có thể được đặt trong `errno` bởi system call UNIX truyền thống.

Vì mỗi tham chiếu đến `errno` trong một chương trình đa luồng mang theo chi phí của một lệnh gọi hàm, các chương trình ví dụ của chúng ta không trực tiếp gán giá trị trả về của một hàm Pthreads cho `errno`. Thay vào đó, chúng ta sử dụng một biến trung gian và dùng hàm chẩn đoán `errExitEN()` (Mục 3.5.2), như sau:

```
pthread_t *thread;
int s;
s = pthread_create(&thread, NULL, func, &arg);
if (s != 0)
 errExitEN(s, "pthread_create");
```

### **Biên dịch chương trình Pthreads**

Trên Linux, các chương trình sử dụng API Pthreads phải được biên dịch với tùy chọn `cc –pthread`. Các hiệu ứng của tùy chọn này bao gồm:

-  Macro preprocessor `_REENTRANT` được định nghĩa. Điều này làm cho các khai báo của một số hàm reentrant được hiển thị.
-  Chương trình được liên kết với thư viện `libpthread` (tương đương với `–lpthread`).

Các tùy chọn chính xác để biên dịch một chương trình đa luồng thay đổi trên các triển khai (và trình biên dịch). Một số triển khai khác (ví dụ: Tru64) cũng sử dụng `cc –pthread`; Solaris và HP-UX sử dụng `cc –mt`.

## **29.3 Tạo Thread**

Khi một chương trình được khởi động, process kết quả bao gồm một thread duy nhất, được gọi là thread khởi tạo hoặc main thread. Trong mục này, chúng ta xem xét cách tạo thêm các thread.

Hàm `pthread_create()` tạo ra một thread mới.

```
#include <pthread.h>
int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
 void *(*start)(void *), void *arg);
                    Returns 0 on success, or a positive error number on error
```

Thread mới bắt đầu thực thi bằng cách gọi hàm được xác định bởi `start` với đối số `arg` (tức là `start(arg)`). Thread gọi `pthread_create()` tiếp tục thực thi với câu lệnh tiếp theo sau lệnh gọi. (Hành vi này giống với hàm wrapper glibc cho system call `clone()` được mô tả trong Mục 28.2.)

Đối số `arg` được khai báo là `void *`, nghĩa là chúng ta có thể truyền pointer đến bất kỳ loại đối tượng nào cho hàm `start`. Thông thường, `arg` trỏ đến một biến global hoặc heap, nhưng nó cũng có thể được chỉ định là NULL. Nếu chúng ta cần truyền nhiều đối số cho `start`, thì `arg` có thể được chỉ định là một pointer đến một cấu trúc chứa các đối số dưới dạng các trường riêng biệt. Với casting phù hợp, chúng ta thậm chí có thể chỉ định `arg` là một `int`.

> Nói chính xác, các tiêu chuẩn C không định nghĩa kết quả của việc cast `int` thành `void *` và ngược lại. Tuy nhiên, hầu hết các trình biên dịch C cho phép các thao tác này và tạo ra kết quả mong muốn; tức là `int j == (int) ((void *) j)`.

Giá trị trả về của `start` cũng có kiểu `void *`, và nó có thể được sử dụng theo cách giống như đối số `arg`. Chúng ta sẽ thấy cách giá trị này được sử dụng khi mô tả hàm `pthread_join()` bên dưới.

> Cần thận trọng khi sử dụng một integer đã được cast làm giá trị trả về của hàm start của một thread. Lý do là `PTHREAD_CANCELED`, giá trị được trả về khi một thread bị hủy (xem Chương [32\)](#page-54-0), thường là một số giá trị integer được xác định bởi triển khai được cast thành `void *`. Nếu hàm start của một thread trả về cùng giá trị integer đó, thì, đối với một thread khác đang thực hiện `pthread_join()`, nó sẽ
>
> sai lầm cho rằng thread đã bị hủy. Trong một ứng dụng sử dụng thread cancellation và chọn trả về các giá trị integer đã cast từ các hàm start của thread, chúng ta phải đảm bảo rằng một thread kết thúc bình thường không trả về một integer có giá trị khớp với `PTHREAD_CANCELED` trên triển khai Pthreads đó. Một ứng dụng di động cần đảm bảo rằng các thread kết thúc bình thường không trả về các giá trị integer khớp với `PTHREAD_CANCELED` trên bất kỳ triển khai nào mà ứng dụng sẽ chạy.

Đối số `thread` trỏ đến một buffer kiểu `pthread_t` vào đó định danh duy nhất cho thread này được sao chép trước khi `pthread_create()` trả về. Định danh này có thể được sử dụng trong các lệnh gọi Pthreads sau đó để tham chiếu đến thread.

> SUSv3 ghi chú rõ ràng rằng triển khai không cần khởi tạo buffer được trỏ bởi `thread` trước khi thread mới bắt đầu thực thi; tức là, thread mới có thể bắt đầu chạy trước khi `pthread_create()` trả về cho caller của nó. Nếu thread mới cần lấy ID của chính nó, thì nó phải làm như vậy bằng cách sử dụng `pthread_self()` (được mô tả trong Mục [29.5\)](#page-7-0).

Đối số `attr` là một pointer đến đối tượng `pthread_attr_t` chỉ định các thuộc tính khác nhau cho thread mới. Chúng ta sẽ nói thêm về các thuộc tính này trong Mục [29.8.](#page-11-0) Nếu `attr` được chỉ định là NULL, thì thread được tạo với các thuộc tính mặc định khác nhau, và đây là điều chúng ta sẽ làm trong hầu hết các chương trình ví dụ trong cuốn sách này.

Sau một lệnh gọi `pthread_create()`, chương trình không có đảm bảo nào về thread nào sẽ được lên lịch sử dụng CPU tiếp theo (trên hệ thống đa bộ xử lý, cả hai thread có thể đồng thời thực thi trên các CPU khác nhau). Các chương trình ngầm dựa vào một thứ tự lên lịch cụ thể đều dễ bị mắc phải cùng loại race condition mà chúng ta đã mô tả trong Mục 24.4. Nếu chúng ta cần thực thi một thứ tự thực thi cụ thể, chúng ta phải sử dụng một trong các kỹ thuật synchronization được mô tả trong Chương [30](#page-14-0).

# **29.4 Kết thúc Thread**

Việc thực thi của một thread kết thúc theo một trong các cách sau:

-  Hàm start của thread thực hiện lệnh return chỉ định giá trị trả về cho thread.
-  Thread gọi `pthread_exit()` (được mô tả bên dưới).
-  Thread bị hủy bằng cách sử dụng `pthread_cancel()` (được mô tả trong Mục [32.1\)](#page-54-1).
-  Bất kỳ thread nào gọi `exit()`, hoặc main thread thực hiện lệnh return (trong hàm `main()`), điều này khiến tất cả các thread trong process kết thúc ngay lập tức.

Hàm `pthread_exit()` kết thúc thread đang gọi và chỉ định giá trị trả về có thể được lấy trong thread khác bằng cách gọi `pthread_join()`.

```
include <pthread.h>
void pthread_exit(void *retval);
```

Gọi `pthread_exit()` tương đương với việc thực hiện lệnh return trong hàm start của thread, với sự khác biệt là `pthread_exit()` có thể được gọi từ bất kỳ hàm nào đã được gọi bởi hàm start của thread.

Đối số `retval` chỉ định giá trị trả về cho thread. Giá trị được trỏ bởi `retval` không nên nằm trên stack của thread, vì nội dung của stack đó trở nên không xác định khi thread kết thúc. (Ví dụ: vùng virtual memory của process đó có thể được tái sử dụng ngay lập tức bởi stack cho một thread mới.) Câu lệnh tương tự áp dụng cho giá trị được đưa vào câu lệnh return trong hàm start của thread.

<span id="page-7-1"></span>Nếu main thread gọi `pthread_exit()` thay vì gọi `exit()` hoặc thực hiện return, thì các thread khác tiếp tục thực thi.

## <span id="page-7-0"></span>**29.5 Thread ID**

Mỗi thread trong một process được xác định duy nhất bởi một thread ID. ID này được trả về cho caller của `pthread_create()`, và một thread có thể lấy ID của chính nó bằng cách sử dụng `pthread_self()`.

```
include <pthread.h>
pthread_t pthread_self(void);
                                      Returns the thread ID of the calling thread
```

Thread ID hữu ích trong các ứng dụng vì những lý do sau:

-  Các hàm Pthreads khác nhau sử dụng thread ID để xác định thread mà chúng sẽ tác động. Ví dụ về các hàm như vậy bao gồm `pthread_join()`, `pthread_detach()`, `pthread_cancel()`, và `pthread_kill()`, tất cả đều được mô tả trong chương này và các chương tiếp theo.
-  Trong một số ứng dụng, có thể hữu ích khi gắn thẻ các cấu trúc dữ liệu động với ID của một thread cụ thể. Điều này có thể phục vụ để xác định thread đã tạo hoặc "sở hữu" một cấu trúc dữ liệu, hoặc có thể được sử dụng bởi một thread để xác định một thread cụ thể sẽ làm điều gì đó với cấu trúc dữ liệu đó sau này.

Hàm `pthread_equal()` cho phép chúng ta kiểm tra xem hai thread ID có giống nhau hay không.

```
include <pthread.h>
int pthread_equal(pthread_t t1, pthread_t t2);
                        Returns nonzero value if t1 and t2 are equal, otherwise 0
```

Ví dụ, để kiểm tra xem ID của thread đang gọi có khớp với thread ID được lưu trong biến `tid` hay không, chúng ta có thể viết như sau:

```
if (pthread_equal(tid, pthread_self())
 printf("tid matches self\n");
```

Hàm `pthread_equal()` là cần thiết vì kiểu dữ liệu `pthread_t` phải được coi là opaque data. Trên Linux, `pthread_t` tình cờ được định nghĩa là `unsigned long`, nhưng trên các triển khai khác, nó có thể là pointer hoặc cấu trúc.

```
In NPTL, pthread_t is actually a pointer that has been cast to unsigned long.
```

SUSv3 không yêu cầu `pthread_t` được triển khai như một kiểu scalar; nó có thể là một cấu trúc. Do đó, chúng ta không thể sử dụng code như sau một cách di động để hiển thị thread ID (mặc dù nó hoạt động trên nhiều triển khai, bao gồm Linux, và đôi khi hữu ích cho mục đích gỡ lỗi):

```
pthread_t thr;
printf("Thread ID = %ld\n", (long) thr); /* WRONG! */
```

Trong các triển khai threading Linux, thread ID là duy nhất trên các process. Tuy nhiên, điều này không nhất thiết đúng trên các triển khai khác, và SUSv3 ghi chú rõ ràng rằng một ứng dụng không thể sử dụng thread ID một cách di động để xác định một thread trong process khác. SUSv3 cũng lưu ý rằng một triển khai được phép tái sử dụng thread ID sau khi một thread đã kết thúc đã được join bằng `pthread_join()` hoặc sau khi một detached thread đã kết thúc. (Chúng ta giải thích `pthread_join()` trong mục tiếp theo, và detached thread trong Mục [29.7.](#page-10-0))

> POSIX thread ID không giống với thread ID được trả về bởi system call `gettid()` đặc thù Linux. POSIX thread ID được gán và duy trì bởi triển khai threading. Thread ID được trả về bởi `gettid()` là một số (tương tự như process ID) được gán bởi kernel. Mặc dù mỗi POSIX thread có một kernel thread ID duy nhất trong triển khai threading NPTL của Linux, nhưng một ứng dụng thường không cần biết về kernel ID (và sẽ không di động nếu phụ thuộc vào việc biết chúng).

## **29.6 Join với một Thread đã Kết thúc**

Hàm `pthread_join()` chờ thread được xác định bởi `thread` kết thúc. (Nếu thread đó đã kết thúc, `pthread_join()` trả về ngay lập tức.) Thao tác này được gọi là joining.

```
include <pthread.h>
int pthread_join(pthread_t thread, void **retval);
                      Returns 0 on success, or a positive error number on error
```

Nếu `retval` là một pointer khác NULL, thì nó nhận được một bản sao giá trị trả về của thread đã kết thúc—tức là giá trị được chỉ định khi thread thực hiện return hoặc gọi `pthread_exit()`.

Gọi `pthread_join()` cho một thread ID đã được join trước đó có thể dẫn đến hành vi không thể đoán trước; ví dụ, nó có thể thay vào đó join với một thread được tạo sau đó tình cờ tái sử dụng cùng thread ID.

Nếu một thread không bị detach (xem Mục [29.7\)](#page-10-0), thì chúng ta phải join với nó bằng cách sử dụng `pthread_join()`. Nếu chúng ta không làm điều này, thì khi thread kết thúc, nó tạo ra tương đương thread của một zombie process (Mục 26.2). Ngoài việc lãng phí tài nguyên hệ thống, nếu đủ thread zombie tích lũy, chúng ta sẽ không thể tạo thêm thread.

Nhiệm vụ mà `pthread_join()` thực hiện cho các thread tương tự như nhiệm vụ được thực hiện bởi `waitpid()` cho các process. Tuy nhiên, có một số sự khác biệt đáng chú ý:

-  Các thread là peer. Bất kỳ thread nào trong một process đều có thể sử dụng `pthread_join()` để join với bất kỳ thread nào khác trong process. Ví dụ, nếu thread A tạo thread B, và thread B tạo thread C, thì thread A có thể join với thread C, hoặc ngược lại. Điều này khác với mối quan hệ phân cấp giữa các process. Khi một process cha tạo một process con bằng `fork()`, chỉ có process đó mới có thể `wait()` trên process con đó. Không có mối quan hệ như vậy giữa thread gọi `pthread_create()` và thread mới kết quả.
-  Không có cách nào nói "join với bất kỳ thread nào" (đối với process, chúng ta có thể làm điều này bằng lệnh gọi `waitpid(–1, &status, options)`); cũng không có cách thực hiện non-blocking join (tương tự như cờ `WNOHANG` của `waitpid()`). Có những cách để đạt được chức năng tương tự bằng cách sử dụng condition variable; chúng ta trình bày ví dụ trong Mục [30.2.4](#page-31-0).

Hạn chế mà `pthread_join()` chỉ có thể join với một thread ID cụ thể là có chủ đích. Ý tưởng là một chương trình chỉ nên join với các thread mà nó "biết" về. Vấn đề với thao tác "join với bất kỳ thread nào" xuất phát từ thực tế là không có phân cấp thread, vì vậy thao tác như vậy thực sự có thể join với bất kỳ thread nào, kể cả thread được tạo riêng bởi một library function. (Kỹ thuật condition-variable mà chúng ta trình bày trong Mục [30.2.4](#page-31-0) cho phép một thread chỉ join với bất kỳ thread nào khác mà nó biết về.) Do đó, thư viện sẽ không còn có thể join với thread đó để lấy trạng thái của nó, và nó sẽ cố gắng sai lầm join với một thread ID đã được join. Nói cách khác, thao tác "join với bất kỳ thread nào" không tương thích với thiết kế chương trình module.

## **Chương trình ví dụ**

Chương trình trong Listing 29-1 tạo một thread khác rồi join với nó.

**Listing 29-1:** Một chương trình đơn giản sử dụng Pthreads

––––––––––––––––––––––––––––––––––––––––––––––––––– **threads/simple\_thread.c** #include <pthread.h> #include "tlpi\_hdr.h" static void \* threadFunc(void \*arg) { char \*s = (char \*) arg; printf("%s", s); return (void \*) strlen(s); }

```
int
main(int argc, char *argv[])
{
 pthread_t t1;
 void *res;
 int s;
 s = pthread_create(&t1, NULL, threadFunc, "Hello world\n");
 if (s != 0)
 errExitEN(s, "pthread_create");
 printf("Message from main()\n");
 s = pthread_join(t1, &res);
 if (s != 0)
 errExitEN(s, "pthread_join");
 printf("Thread returned %ld\n", (long) res);
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––– threads/simple_thread.c
```

Khi chạy chương trình trong Listing 29-1, chúng ta thấy như sau:

```
$ ./simple_thread
Message from main()
Hello world
Thread returned 12
```

Tùy thuộc vào cách hai thread được lên lịch, thứ tự của hai dòng đầu ra đầu tiên có thể bị đảo ngược.

# <span id="page-10-0"></span>**29.7 Detach Thread**

Mặc định, một thread có thể joinable, nghĩa là khi nó kết thúc, thread khác có thể lấy trạng thái trả về của nó bằng cách sử dụng `pthread_join()`. Đôi khi, chúng ta không quan tâm đến trạng thái trả về của thread; chúng ta chỉ muốn hệ thống tự động dọn dẹp và xóa thread khi nó kết thúc. Trong trường hợp này, chúng ta có thể đánh dấu thread là detached, bằng cách gọi `pthread_detach()` chỉ định định danh của thread trong `thread`.

```
#include <pthread.h>
int pthread_detach(pthread_t thread);
                      Returns 0 on success, or a positive error number on error
```

Như ví dụ về việc sử dụng `pthread_detach()`, một thread có thể detach chính nó bằng lệnh gọi sau:

```
pthread_detach(pthread_self());
```

Khi một thread đã được detach, không còn có thể sử dụng `pthread_join()` để lấy trạng thái trả về của nó, và thread không thể được làm joinable lại.

Detach một thread không làm cho nó miễn nhiễm với lệnh gọi `exit()` trong một thread khác hoặc lệnh return trong main thread. Trong trường hợp như vậy, tất cả các thread trong process đều bị kết thúc ngay lập tức, bất kể chúng là joinable hay detached. Nói cách khác, `pthread_detach()` chỉ kiểm soát điều gì xảy ra sau khi một thread kết thúc, không phải cách thức hoặc thời điểm nó kết thúc.

## <span id="page-11-0"></span>**29.8 Thuộc tính Thread**

<span id="page-11-2"></span>Chúng ta đã đề cập trước đó rằng đối số `attr` của `pthread_create()`, có kiểu `pthread_attr_t`, có thể được sử dụng để chỉ định các thuộc tính được sử dụng trong việc tạo thread mới. Chúng ta sẽ không đi vào chi tiết của các thuộc tính này (để biết chi tiết đó, hãy xem các tài liệu tham khảo được liệt kê ở cuối chương này) hoặc hiển thị nguyên mẫu của các hàm Pthreads khác nhau có thể được sử dụng để thao tác đối tượng `pthread_attr_t`. Chúng ta chỉ đề cập rằng các thuộc tính này bao gồm thông tin như vị trí và kích thước của stack thread, chính sách và độ ưu tiên scheduling của thread (tương tự như chính sách và độ ưu tiên realtime scheduling của process được mô tả trong Mục 35.2 và 35.3), và liệu thread có phải là joinable hay detached.

Như ví dụ về việc sử dụng thuộc tính thread, code được hiển thị trong [Listing 29-2](#page-11-1) tạo một thread mới được làm detached tại thời điểm tạo thread (thay vì sau đó, bằng cách sử dụng `pthread_detach()`). Code này trước tiên khởi tạo một cấu trúc thuộc tính thread với các giá trị mặc định, đặt thuộc tính cần thiết để tạo một detached thread, sau đó tạo một thread mới sử dụng cấu trúc thuộc tính thread. Sau khi thread đã được tạo, đối tượng thuộc tính không còn cần thiết nữa, và vì vậy được hủy.

<span id="page-11-1"></span>**Listing 29-2:** Tạo thread với thuộc tính detached

```
––––––––––––––––––––––––––––––––––––––––––––– fromthreads/detached_attrib.c
 pthread_t thr;
 pthread_attr_t attr;
 int s;
 s = pthread_attr_init(&attr); /* Assigns default values */
 if (s != 0)
 errExitEN(s, "pthread_attr_init");
 s = pthread_attr_setdetachstate(&attr, PTHREAD_CREATE_DETACHED);
 if (s != 0)
 errExitEN(s, "pthread_attr_setdetachstate");
 s = pthread_create(&thr, &attr, threadFunc, (void *) 1);
 if (s != 0)
 errExitEN(s, "pthread_create");
 s = pthread_attr_destroy(&attr); /* No longer needed */
 if (s != 0)
 errExitEN(s, "pthread_attr_destroy");
––––––––––––––––––––––––––––––––––––––––––––– fromthreads/detached_attrib.c
```

## **29.9 Threads So với Processes**

Trong mục này, chúng ta ngắn gọn xem xét một số yếu tố có thể ảnh hưởng đến sự lựa chọn của chúng ta về việc triển khai ứng dụng dưới dạng một nhóm thread hay một nhóm process. Chúng ta bắt đầu bằng cách xem xét các ưu điểm của phương pháp đa luồng:

-  Chia sẻ dữ liệu giữa các thread là dễ dàng. Ngược lại, chia sẻ dữ liệu giữa các process đòi hỏi nhiều công việc hơn (ví dụ: tạo một shared memory segment hoặc sử dụng pipe).
-  Tạo thread nhanh hơn tạo process; thời gian chuyển ngữ cảnh (context-switch) có thể thấp hơn cho thread so với process.

Sử dụng thread có thể có một số nhược điểm so với sử dụng process:

-  Khi lập trình với thread, chúng ta cần đảm bảo rằng các hàm chúng ta gọi là thread-safe hoặc được gọi theo cách thread-safe. (Chúng ta mô tả khái niệm thread safety trong Mục [31.1.](#page-38-0)) Các ứng dụng đa tiến trình không cần lo lắng về điều này.
-  Một bug trong một thread (ví dụ: sửa đổi bộ nhớ qua một pointer không chính xác) có thể làm hỏng tất cả các thread trong process, vì chúng chia sẻ cùng address space và các thuộc tính khác. Ngược lại, các process được cách ly hơn với nhau.
-  Mỗi thread cạnh tranh để sử dụng virtual address space hữu hạn của host process. Đặc biệt, stack của mỗi thread và thread-specific data (hoặc thread-local storage) tiêu thụ một phần của virtual address space của process, do đó không có sẵn cho các thread khác. Mặc dù virtual address space có sẵn lớn (ví dụ: thường là 3 GB trên x86-32), yếu tố này có thể là một hạn chế đáng kể đối với các process sử dụng số lượng lớn thread hoặc các thread yêu cầu lượng bộ nhớ lớn. Ngược lại, các process riêng biệt có thể sử dụng toàn bộ phạm vi virtual memory có sẵn (tùy thuộc vào các hạn chế của RAM và swap space).

Sau đây là một số điểm khác có thể ảnh hưởng đến sự lựa chọn của chúng ta giữa thread và process:

-  Xử lý signal trong một ứng dụng đa luồng đòi hỏi thiết kế cẩn thận. (Như một nguyên tắc chung, thường nên tránh sử dụng signal trong các chương trình đa luồng.) Chúng ta nói thêm về thread và signal trong Mục [33.2](#page-65-0).
-  Trong một ứng dụng đa luồng, tất cả các thread phải chạy cùng một chương trình (mặc dù có thể trong các hàm khác nhau). Trong một ứng dụng đa tiến trình, các process khác nhau có thể chạy các chương trình khác nhau.
-  Bên cạnh dữ liệu, các thread cũng chia sẻ một số thông tin khác (ví dụ: file descriptor, signal disposition, thư mục làm việc hiện tại, và user và group ID). Điều này có thể là ưu điểm hoặc nhược điểm, tùy thuộc vào ứng dụng.

# **29.10 Tóm tắt**

<span id="page-12-0"></span>Trong một process đa luồng, nhiều thread đang đồng thời thực thi cùng một chương trình. Tất cả các thread chia sẻ cùng các biến global và heap, nhưng mỗi thread có một stack riêng cho các biến cục bộ. Các thread trong một process cũng chia sẻ một số thuộc tính khác, bao gồm process ID, các file descriptor đang mở, signal disposition, thư mục làm việc hiện tại, và giới hạn tài nguyên.

Sự khác biệt chính giữa thread và process là việc chia sẻ thông tin dễ dàng hơn mà các thread cung cấp, và đây là lý do chính mà một số thiết kế ứng dụng phù hợp hơn với thiết kế đa luồng so với thiết kế đa tiến trình. Các thread cũng có thể cung cấp hiệu suất tốt hơn cho một số thao tác (ví dụ: tạo thread nhanh hơn tạo process), nhưng yếu tố này thường là thứ yếu trong việc ảnh hưởng đến sự lựa chọn giữa thread và process.

Các thread được tạo bằng `pthread_create()`. Mỗi thread sau đó có thể độc lập kết thúc bằng cách sử dụng `pthread_exit()`. (Nếu bất kỳ thread nào gọi `exit()`, thì tất cả các thread kết thúc ngay lập tức.) Trừ khi một thread đã được đánh dấu là detached (ví dụ: thông qua lệnh gọi `pthread_detach()`), nó phải được join bởi một thread khác bằng cách sử dụng `pthread_join()`, điều này trả về trạng thái kết thúc của thread đã được join.

### **Thông tin thêm**

[Butenhof, 1996] cung cấp một trình bày về Pthreads vừa dễ đọc vừa toàn diện. [Robbins & Robbins, 2003] cũng cung cấp coverage tốt về Pthreads. [Tanenbaum, 2007] cung cấp một giới thiệu lý thuyết hơn về các khái niệm thread, bao gồm các chủ đề như mutex, critical region, conditional variable, và phát hiện và tránh deadlock. [Vahalia, 1996] cung cấp nền tảng về triển khai của thread.

## **29.11 Bài tập**

**29-1.** Những kết quả có thể nào có thể xảy ra nếu một thread thực thi code sau:

```
pthread_join(pthread_self(), NULL);
```

Viết một chương trình để xem thực sự điều gì xảy ra trên Linux. Nếu chúng ta có một biến, `tid`, chứa thread ID, làm thế nào một thread có thể ngăn chính nó thực hiện một lệnh gọi `pthread_join(tid, NULL)` tương đương với câu lệnh trên?

**29-2.** Ngoài sự vắng mặt của kiểm tra lỗi và các khai báo biến và cấu trúc khác nhau, vấn đề gì với chương trình sau?

```
static void *
threadFunc(void *arg)
{
 struct someStruct *pbuf = (struct someStruct *) arg;
 /* Do some work with structure pointed to by 'pbuf' */
}
int
main(int argc, char *argv[])
{
 struct someStruct buf;
 pthread_create(&thr, NULL, threadFunc, (void *) &buf);
 pthread_exit(NULL);
}
```
