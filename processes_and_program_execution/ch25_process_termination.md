## Chương 25
# Kết Thúc Process (Process Termination)

Chương này mô tả điều gì xảy ra khi một process kết thúc. Chúng ta bắt đầu bằng cách mô tả cách dùng `exit()` và `_exit()` để kết thúc process, sau đó thảo luận về exit handler để tự động thực hiện dọn dẹp khi process gọi `exit()`. Kết thúc bằng xem xét một số tương tác giữa `fork()`, stdio buffer, và `exit()`.

---

## 25.1 Kết Thúc Process: _exit() và exit()

Process có thể kết thúc theo hai cách chung. Một là **abnormal termination** (kết thúc bất thường), do signal có default action là kết thúc process (với hoặc không có core dump). Cách còn lại là process kết thúc bình thường bằng system call `_exit()`.

```c
#include <unistd.h>
void _exit(int status);
```

Đối số `status` cho `_exit()` định nghĩa termination status của process, có sẵn cho process cha khi gọi `wait()`. Dù được định nghĩa là `int`, chỉ 8 bit thấp nhất của `status` thực sự được cung cấp cho process cha. Theo quy ước, termination status 0 chỉ ra process hoàn thành thành công, còn giá trị khác 0 chỉ ra kết thúc không thành công. SUSv3 quy định hai hằng số, `EXIT_SUCCESS` (0) và `EXIT_FAILURE` (1).

Process luôn kết thúc thành công bởi `_exit()` (tức là `_exit()` không bao giờ trả về).

Programs thường không gọi `_exit()` trực tiếp mà thay vào đó gọi hàm thư viện `exit()`, thực hiện các hành động khác nhau trước khi gọi `_exit()`.

```c
#include <stdlib.h>
void exit(int status);
```

Các hành động được thực hiện bởi `exit()`:

- Exit handler (hàm đăng ký với `atexit()` và `on_exit()`) được gọi, theo thứ tự ngược với đăng ký (Mục 25.3).
- Các stdio stream buffer được flush.
- System call `_exit()` được gọi với giá trị `status`.

Một cách khác để process kết thúc là return từ `main()`, hoặc rõ ràng, hoặc ngầm định. Thực hiện `return n` thường tương đương với gọi `exit(n)`.

---

## 25.2 Chi Tiết về Kết Thúc Process

Trong cả normal và abnormal termination, các hành động sau xảy ra:

- Open file descriptor, directory stream, message catalog descriptor, và conversion descriptor được đóng.
- Hệ quả của việc đóng file descriptor: các file lock (Chương 55) do process này giữ được giải phóng.
- Mọi System V shared memory segment được đính kèm sẽ bị detach, và `shm_nattch` counter cho mỗi segment giảm đi một.
- Với mỗi System V semaphore có `semadj` value được process đặt, giá trị `semadj` đó được cộng vào semaphore value.
- Nếu đây là controlling process cho một controlling terminal, signal `SIGHUP` được gửi đến mỗi process trong foreground process group của terminal đó, và terminal bị tách khỏi session.
- Mọi POSIX named semaphore đang mở trong process gọi được đóng như thể `sem_close()` được gọi.
- Mọi POSIX message queue đang mở trong process gọi được đóng như thể `mq_close()` được gọi.
- Nếu hệ quả của việc process này thoát làm một process group trở nên orphaned và có process bị stopped, tất cả process trong group sẽ nhận `SIGHUP` rồi `SIGCONT`.
- Mọi memory lock được thiết lập bởi process này qua `mlock()` hoặc `mlockall()` (Mục 50.2) được xóa.
- Mọi memory mapping được thiết lập bởi process này qua `mmap()` được unmap.

---

## 25.3 Exit Handler

Đôi khi ứng dụng cần tự động thực hiện một số thao tác khi kết thúc process. **Exit handler** là hàm do lập trình viên cung cấp, được đăng ký tại một thời điểm nào đó trong vòng đời process và tự động được gọi trong quá trình normal process termination qua `exit()`. Exit handler không được gọi nếu chương trình gọi `_exit()` trực tiếp hoặc nếu process bị kết thúc bất thường bởi signal.

### Đăng ký exit handler

Cách đầu tiên để đăng ký exit handler, được quy định trong SUSv3, là dùng `atexit()`.

```c
#include <stdlib.h>
int atexit(void (*func)(void));
                                      Returns 0 on success, or nonzero on error
```

`atexit()` thêm `func` vào danh sách hàm được gọi khi process kết thúc. Hàm `func` không nhận đối số và không trả về giá trị:

```c
void
func(void)
{
    /* Thực hiện một số hành động */
}
```

Có thể đăng ký nhiều exit handler (và thậm chí cùng một exit handler nhiều lần). Khi chương trình gọi `exit()`, các hàm này được gọi theo thứ tự ngược với đăng ký.

Child process tạo qua `fork()` kế thừa bản sao của exit handler registration của parent. Khi process thực hiện `exec()`, tất cả exit handler registration bị xóa.

Exit handler được đăng ký với `atexit()` có một số hạn chế: không biết status nào được truyền cho `exit()`, và không thể chỉ định đối số cho exit handler khi gọi.

Để giải quyết các hạn chế này, glibc cung cấp phương thức đăng ký exit handler thay thế (không chuẩn): `on_exit()`.

```c
#define _BSD_SOURCE
#include <stdlib.h>
int on_exit(void (*func)(int, void *), void *arg);
                                    Returns 0 on success, or nonzero on error
```

`func` là pointer đến hàm nhận hai đối số:
- `status`: đối số được truyền cho `exit()`
- `arg`: bản sao của đối số `arg` được cung cấp cho `on_exit()` lúc đăng ký

Mặc dù linh hoạt hơn `atexit()`, `on_exit()` nên tránh trong chương trình cần portable vì không được chuẩn hóa.

**Listing 25-1:** Sử dụng exit handler

```c
–––––––––––––––––––––––––––––––––––––––––––––––––– procexec/exit_handlers.c
#define _BSD_SOURCE
#include <stdlib.h>
#include "tlpi_hdr.h"
static void
atexitFunc1(void)
{
    printf("atexit function 1 called\n");
}
static void
atexitFunc2(void)
{
    printf("atexit function 2 called\n");
}
static void
onexitFunc(int exitStatus, void *arg)
{
    printf("on_exit function called: status=%d, arg=%ld\n",
           exitStatus, (long) arg);
}
int
main(int argc, char *argv[])
{
    if (on_exit(onexitFunc, (void *) 10) != 0)  fatal("on_exit 1");
    if (atexit(atexitFunc1) != 0)               fatal("atexit 1");
    if (atexit(atexitFunc2) != 0)               fatal("atexit 2");
    if (on_exit(onexitFunc, (void *) 20) != 0)  fatal("on_exit 2");
    exit(2);
}
–––––––––––––––––––––––––––––––––––––––––––––––––– procexec/exit_handlers.c
```

Kết quả chạy:
```
$ ./exit_handlers
on_exit function called: status=2, arg=20
atexit function 2 called
atexit function 1 called
on_exit function called: status=2, arg=10
```

---

## 25.4 Tương Tác Giữa fork(), stdio Buffer, và _exit()

Khi chạy chương trình trong Listing 25-2 với stdout hướng đến terminal, ta thấy kết quả bình thường:

```
$ ./fork_stdio_buf
Hello world
Ciao
```

Tuy nhiên, khi redirect stdout sang file:

```
$ ./fork_stdio_buf > a
$ cat a
Ciao
Hello world
Hello world
```

Có hai điều lạ: dòng do `printf()` viết xuất hiện hai lần, và output của `write()` xuất hiện trước `printf()`.

**Listing 25-2:** Tương tác của `fork()` và stdio buffering

```c
––––––––––––––––––––––––––––––––––––––––––––––––– procexec/fork_stdio_buf.c
#include "tlpi_hdr.h"
int
main(int argc, char *argv[])
{
    printf("Hello world\n");
    write(STDOUT_FILENO, "Ciao\n", 5);
    if (fork() == -1)
        errExit("fork");
    /* Cả child lẫn parent tiếp tục thực thi ở đây */
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––– procexec/fork_stdio_buf.c
```

**Giải thích**: stdio buffer được duy trì trong user-space memory của process. Khi `fork()`, các buffer này được nhân đôi trong child. Khi stdout hướng đến terminal, nó là line-buffered và chuỗi kết thúc bằng newline xuất hiện ngay. Nhưng khi redirect sang file, stdout là block-buffered — chuỗi của `printf()` vẫn còn trong buffer của parent lúc `fork()` và được nhân đôi sang child. Khi cả parent lẫn child gọi `exit()`, cả hai đều flush stdio buffer, dẫn đến kết quả trùng lặp.

**Cách ngăn chặn**:
- Dùng `fflush()` để flush stdio buffer trước lời gọi `fork()`.
- Thay vì gọi `exit()`, child gọi `_exit()` để không flush stdio buffer. Đây là nguyên tắc chung: trong ứng dụng tạo child process, thường chỉ một process (thường là parent) kết thúc qua `exit()`, các process khác kết thúc qua `_exit()`.

Output của `write()` không xuất hiện hai lần vì `write()` chuyển dữ liệu trực tiếp đến kernel buffer — không bị nhân đôi trong `fork()`.

---

## 25.5 Tóm Tắt

Process có thể kết thúc bất thường hoặc bình thường. Kết thúc bất thường xảy ra khi nhận một số signal nhất định.

Kết thúc bình thường thực hiện bằng cách gọi `_exit()` hoặc thường hơn là `exit()`, được xây dựng trên `_exit()`. Cả hai đều nhận đối số integer có 8 bit ít có ý nghĩa nhất định nghĩa termination status. Theo quy ước, status 0 chỉ kết thúc thành công, khác 0 chỉ kết thúc không thành công.

Gọi `exit()` còn khiến exit handler đăng ký bằng `atexit()` và `on_exit()` được gọi (theo thứ tự ngược với đăng ký), và stdio buffer được flush.

---

## 25.6 Bài Tập

**25-1.** Nếu child process gọi `exit(-1)`, exit status nào (được trả về bởi `WEXITSTATUS()`) mà parent sẽ thấy?
