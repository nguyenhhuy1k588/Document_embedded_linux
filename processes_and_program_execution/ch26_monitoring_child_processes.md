## Chương 26
# Giám Sát Child Process (Monitoring Child Processes)

Trong nhiều thiết kế ứng dụng, parent process cần biết khi nào một trong các child process của nó thay đổi trạng thái — khi child kết thúc hoặc bị dừng bởi signal. Chương này mô tả hai kỹ thuật để giám sát child process: system call `wait()` (và các biến thể) và việc sử dụng signal `SIGCHLD`.

---

## 26.1 Chờ Child Process

### 26.1.1 System Call wait()

System call `wait()` chờ một trong các child của process gọi kết thúc và trả về termination status của child đó trong buffer được trỏ bởi `status`.

```c
#include <sys/wait.h>
pid_t wait(int *status);
                          Returns process ID of terminated child, or –1 on error
```

`wait()` thực hiện các việc sau:

1. Nếu chưa có child nào (chưa được wait) kết thúc, lời gọi block cho đến khi một child kết thúc. Nếu child đã kết thúc trước đó, `wait()` trả về ngay.
2. Nếu `status` không `NULL`, thông tin về cách child kết thúc được trả về trong integer được trỏ bởi `status` (mô tả trong Mục 26.1.3).
3. Kernel cộng thêm CPU time và resource usage của process vào tổng cộng cho tất cả child của parent process này.
4. `wait()` trả về process ID của child đã kết thúc.

Khi lỗi, `wait()` trả về `-1`. Một lỗi có thể xảy ra là process gọi không có child nào (chưa được wait), được chỉ ra bởi `errno` là `ECHILD`. Dùng vòng lặp sau để chờ tất cả child kết thúc:

```c
while ((childPid = wait(NULL)) != -1)
    continue;
if (errno != ECHILD)  /* Lỗi không mong đợi... */
    errExit("wait");
```

### 26.1.2 System Call waitpid()

`wait()` có một số hạn chế mà `waitpid()` được thiết kế để giải quyết:
- Không thể chờ một child cụ thể.
- `wait()` luôn block nếu chưa có child nào kết thúc.
- Không thể theo dõi child bị stopped bởi signal.

```c
#include <sys/wait.h>
pid_t waitpid(pid_t pid, int *status, int options);
                          Returns process ID of child, 0 (see text), or –1 on error
```

Đối số `pid` cho phép chọn child cần chờ:
- `pid > 0`: Chờ child có process ID bằng `pid`.
- `pid == 0`: Chờ bất kỳ child nào trong cùng process group với caller.
- `pid < -1`: Chờ bất kỳ child nào có process group ID bằng giá trị tuyệt đối của `pid`.
- `pid == -1`: Chờ bất kỳ child nào. `wait(&status)` tương đương `waitpid(-1, &status, 0)`.

Đối số `options` là bit mask có thể bao gồm:

- **`WUNTRACED`**: Ngoài thông tin về child đã kết thúc, cũng trả về thông tin khi child bị stopped bởi signal.
- **`WCONTINUED`** (từ Linux 2.6.10): Trả về status về child bị stopped được resume bởi `SIGCONT`.
- **`WNOHANG`**: Nếu chưa có child nào được chỉ định bởi `pid` thay đổi trạng thái, trả về ngay lập tức (poll). Trong trường hợp này, `waitpid()` trả về 0.

### 26.1.3 Giá Trị Wait Status

Giá trị `status` trả về bởi `wait()` và `waitpid()` cho phép phân biệt các sự kiện sau của child:

- Child kết thúc bằng cách gọi `_exit()` (hoặc `exit()`), chỉ định integer exit status.
- Child bị kết thúc bởi delivery của unhandled signal.
- Child bị stopped bởi signal, và `waitpid()` được gọi với flag `WUNTRACED`.
- Child được resume bởi signal `SIGCONT`, và `waitpid()` được gọi với flag `WCONTINUED`.

`<sys/wait.h>` định nghĩa macro chuẩn để phân tích wait status value:

- **`WIFEXITED(status)`**: True nếu child kết thúc bình thường. Dùng `WEXITSTATUS(status)` để lấy exit status.
- **`WIFSIGNALED(status)`**: True nếu child bị kill bởi signal. `WTERMSIG(status)` trả về số hiệu signal, `WCOREDUMP(status)` trả về true nếu có core dump.
- **`WIFSTOPPED(status)`**: True nếu child bị stopped bởi signal. `WSTOPSIG(status)` trả về số hiệu signal.
- **`WIFCONTINUED(status)`**: True nếu child được resume bởi `SIGCONT`.

### 26.1.4 Kết Thúc Process từ Signal Handler

Nếu cần thông báo cho parent biết child kết thúc do signal, signal handler của child nên trước tiên huỷ thiết lập handler, rồi raise lại signal đó:

```c
void
handler(int sig)
{
    /* Thực hiện các bước dọn dẹp */
    signal(sig, SIG_DFL);  /* Huỷ thiết lập handler */
    raise(sig);             /* Raise signal lại */
}
```

### 26.1.5 System Call waitid()

Giống `waitpid()`, `waitid()` trả về status của child process nhưng cung cấp thêm chức năng. Được thêm vào Linux trong kernel 2.6.9.

```c
#include <sys/wait.h>
int waitid(idtype_t idtype, id_t id, siginfo_t *infop, int options);
                              Returns 0 on success or if WNOHANG was specified and
                                there were no children to wait for, or –1 on error
```

`idtype` và `id` chỉ định child(ren) nào cần chờ:
- `P_ALL`: Chờ bất kỳ child nào; `id` bị bỏ qua.
- `P_PID`: Chờ child có process ID bằng `id`.
- `P_PGID`: Chờ bất kỳ child nào có process group ID bằng `id`.

`options` có thể bao gồm: `WEXITED`, `WSTOPPED`, `WCONTINUED`, `WNOHANG`, `WNOWAIT`.

Khi thành công, `waitid()` trả về 0 và structure `siginfo_t` được trỏ bởi `infop` được cập nhật với các trường: `si_code`, `si_pid`, `si_signo`, `si_status`, `si_uid`.

### 26.1.6 System Call wait3() và wait4()

`wait3()` và `wait4()` thực hiện tác vụ tương tự `waitpid()` nhưng còn trả về thông tin resource usage của child đã kết thúc trong structure được trỏ bởi đối số `rusage`. Không được chuẩn hóa trong SUSv3.

---

## 26.2 Orphan và Zombie

**Orphan**: Khi parent của một child kết thúc trước, child đó trở thành orphan và được `init` (process ID 1) nhận nuôi.

**Zombie**: Khi child kết thúc trước khi parent thực hiện `wait()`, process đó được chuyển thành zombie — hầu hết tài nguyên của nó được giải phóng nhưng vẫn còn entry trong process table của kernel (chứa process ID, termination status, và resource usage statistics).

Zombie process không thể bị kill bởi signal, kể cả `SIGKILL`. Điều này đảm bảo parent luôn có thể thực hiện `wait()` sau này.

Khi parent cuối cùng thực hiện `wait()`, kernel xóa zombie. Nếu parent kết thúc mà không thực hiện `wait()`, `init` sẽ nhận nuôi zombie và thực hiện `wait()`, xóa chúng khỏi hệ thống.

Nếu nhiều zombie tích lũy, chúng có thể lấp đầy kernel process table, ngăn tạo process mới. Vì vậy, long-running program (như network server và shell) cần thực hiện `wait()` call để đảm bảo dead child luôn được xóa.

---

## 26.3 Signal SIGCHLD

Kết thúc của child process là sự kiện xảy ra không đồng bộ. Để giám sát child mà không cần block hay polling, ta có thể dùng handler cho signal `SIGCHLD`.

### 26.3.1 Thiết Lập Handler cho SIGCHLD

`SIGCHLD` được gửi đến parent process bất cứ khi nào một trong các child của nó kết thúc. Mặc định, signal này bị bỏ qua nhưng ta có thể bắt nó bằng signal handler.

Vì `SIGCHLD` không được xếp hàng, nếu nhiều child kết thúc nhanh, chỉ một `SIGCHLD` được xếp hàng. Do đó, handler phải lặp vòng để reap tất cả child đã chết:

```c
while (waitpid(-1, NULL, WNOHANG) > 0)
    continue;
```

Handler nên lưu và khôi phục `errno` vì `waitpid()` có thể thay đổi nó:

```c
static void
sigchldHandler(int sig)
{
    int status, savedErrno;
    pid_t childPid;
    savedErrno = errno;
    while ((childPid = waitpid(-1, &status, WNOHANG)) > 0) {
        /* Xử lý child đã kết thúc */
    }
    errno = savedErrno;
}
```

**Lưu ý thiết kế**: Nên thiết lập handler SIGCHLD trước khi tạo bất kỳ child nào, và nên block `SIGCHLD` trước khi tạo child để tránh race condition với `sigsuspend()`.

### 26.3.2 Delivery SIGCHLD cho Stopped Child

Sử dụng flag `SA_NOCLDSTOP` khi thiết lập handler với `sigaction()` kiểm soát liệu `SIGCHLD` có được gửi khi child bị stopped bởi signal không. Nếu bỏ qua flag này, `SIGCHLD` được gửi; nếu có flag, không gửi cho stopped child.

### 26.3.3 Bỏ Qua Dead Child Process

Đặt rõ ràng disposition của `SIGCHLD` thành `SIG_IGN` khiến bất kỳ child nào kết thúc sau đó sẽ bị xóa ngay lập tức khỏi hệ thống thay vì chuyển thành zombie. Status của child bị loại bỏ hoàn toàn.

Lưu ý: Mặc dù default disposition của `SIGCHLD` là bị bỏ qua, việc đặt rõ ràng thành `SIG_IGN` tạo ra hành vi khác — `SIGCHLD` được xử lý đặc biệt.

Flag `SA_NOCLDWAIT` của `sigaction()` tạo ra hành vi tương tự khi disposition của `SIGCHLD` được đặt thành `SIG_IGN`.

---

## 26.4 Tóm Tắt

Dùng `wait()` và `waitpid()` (và các hàm liên quan khác), parent process có thể lấy status của child đã kết thúc và stopped. Status chỉ ra liệu child kết thúc bình thường, kết thúc bất thường, bị stopped bởi signal, hay được resume bởi `SIGCONT`.

Nếu parent của child kết thúc, child trở thành orphan và được `init` nhận nuôi (process ID 1).

Khi child kết thúc, nó trở thành zombie và chỉ bị xóa khỏi hệ thống khi parent gọi `wait()`. Long-running program như shell và daemon nên được thiết kế để luôn reap status của child process chúng tạo.

Cách phổ biến để reap dead child là thiết lập handler cho `SIGCHLD`. Thay thế, nhưng ít portable hơn, process có thể đặt disposition của `SIGCHLD` thành `SIG_IGN` để status của child bị kết thúc bị loại bỏ ngay lập tức.

---

## 26.5 Bài Tập

- **26-1.** Viết chương trình để xác minh rằng khi parent của child kết thúc, lời gọi `getppid()` trả về 1 (process ID của `init`).
- **26-2.** Giả sử có ba process liên quan là grandparent, parent, và child, và grandparent không thực hiện `wait()` ngay sau khi parent kết thúc. Khi nào grandchild được `init` nhận nuôi: sau khi parent kết thúc hay sau khi grandparent thực hiện `wait()`? Viết chương trình để kiểm tra.
- **26-3.** Thay `waitpid()` bằng `waitid()` trong chương trình `child_status.c`.
- **26-4.** Sửa đổi `make_zombie.c` để loại bỏ race condition bằng cách dùng signal để đồng bộ hóa parent và child.
