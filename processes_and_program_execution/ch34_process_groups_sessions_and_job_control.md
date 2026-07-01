## Chương 34
# Process Group, Session và Job Control

Process group và session tạo thành quan hệ phân cấp hai cấp giữa các process: process group là tập hợp các process liên quan, và session là tập hợp các process group liên quan. Process group và session là các abstraction được định nghĩa để hỗ trợ shell job control, cho phép người dùng tương tác chạy lệnh ở foreground hoặc background. Thuật ngữ **job** thường được dùng đồng nghĩa với **process group**.

---

## 34.1 Tổng Quan

**Process group** là tập hợp một hoặc nhiều process chia sẻ cùng process group identifier (PGID). Process group có **leader** — process tạo ra group và có process ID là PGID của group. Process mới kế thừa PGID của parent.

**Session** là tập hợp các process group. Session membership của process được xác định bởi **session ID (SID)**. **Session leader** là process tạo session mới và có process ID là session ID. Process mới kế thừa session ID của parent.

Tất cả process trong một session chia sẻ một **controlling terminal** duy nhất. Controlling terminal được thiết lập khi session leader mở thiết bị terminal lần đầu tiên.

Tại bất kỳ thời điểm nào, một trong các process group trong session là **foreground process group** của terminal, các nhóm khác là **background process group**. Chỉ process trong foreground process group mới có thể đọc input từ controlling terminal. Khi người dùng gõ ký tự tạo signal trên terminal (Control-C → `SIGINT`, Control-\\ → `SIGQUIT`, Control-Z → `SIGTSTP`), signal được gửi đến tất cả thành viên của foreground process group.

Session leader trở thành **controlling process** cho terminal. Khi terminal disconnect xảy ra, kernel gửi `SIGHUP` cho controlling process.

---

## 34.2 Process Group

Process có thể lấy PGID bằng `getpgrp()`.

```c
#include <unistd.h>
pid_t getpgrp(void);
                 Always successfully returns process group ID of calling process
```

Nếu giá trị trả về bởi `getpgrp()` trùng với process ID của caller, process này là leader của process group.

System call `setpgid()` thay đổi process group của process có process ID là `pid` thành `pgid`.

```c
#include <unistd.h>
int setpgid(pid_t pid, pid_t pgid);
                                             Returns 0 on success, or –1 on error
```

Nếu `pid` là 0, thay đổi PGID của process gọi. Nếu `pgid` là 0, PGID được đặt bằng process ID của process được chỉ định bởi `pid`. Nếu `pid` và `pgid` chỉ định cùng process, một process group mới được tạo với process đó là leader.

Các hạn chế khi gọi `setpgid()`:
- `pid` chỉ có thể chỉ định process gọi hoặc một trong các children của nó.
- Process gọi và process được chỉ định bởi `pid`, cũng như target process group, phải là một phần của cùng session.
- `pid` không thể chỉ định session leader.
- Không thể thay đổi PGID của child sau khi child đã thực hiện `exec()`.

**Trong job-control shell**: Để tránh race condition, cả parent lẫn child đều gọi `setpgid()` để thay đổi PGID của child ngay sau `fork()`, và parent bỏ qua lỗi `EACCES` trong lời gọi `setpgid()`.

---

## 34.3 Session

Session membership của process được định nghĩa bởi session ID. `getsid()` trả về session ID của process được chỉ định bởi `pid`.

```c
#define _XOPEN_SOURCE 500
#include <unistd.h>
pid_t getsid(pid_t pid);
                   Returns session ID of specified process, or (pid_t) –1 on error
```

Nếu process gọi **không phải là process group leader**, `setsid()` tạo session mới.

```c
#include <unistd.h>
pid_t setsid(void);
                        Returns session ID of new session, or (pid_t) –1 on error
```

`setsid()` tạo session mới như sau:
- Process gọi trở thành leader của session mới và là leader của process group mới. PGID và session ID đều được đặt bằng process ID.
- Process gọi không có controlling terminal. Mọi kết nối trước đó với controlling terminal bị cắt đứt.

Nếu process gọi là process group leader, `setsid()` thất bại với `EPERM`. Cách đơn giản nhất để đảm bảo điều này không xảy ra là thực hiện `fork()` và để parent thoát trong khi child gọi `setsid()`.

---

## 34.4 Controlling Terminal và Controlling Process

Tất cả process trong session có thể có một controlling terminal duy nhất. Session leader mở terminal device lần đầu (không có flag `O_NOCTTY`) trở thành controlling terminal.

Controlling terminal kế thừa bởi child qua `fork()` và được bảo toàn qua `exec()`. Khi session leader mở controlling terminal, nó đồng thời trở thành **controlling process** cho terminal.

Nếu process có controlling terminal, mở file đặc biệt `/dev/tty` trả về file descriptor cho terminal đó. Nếu không có controlling terminal, mở `/dev/tty` thất bại với `ENXIO`.

`ioctl(fd, TIOCNOTTY)` xóa liên kết của process với controlling terminal.

`ctermid()` trả về pathname tham chiếu đến controlling terminal (thường là `/dev/tty`).

---

## 34.5 Foreground và Background Process Group

Controlling terminal duy trì khái niệm **foreground process group**. Chỉ có một process group trong foreground tại một thời điểm; tất cả process group khác là background.

`tcgetpgrp()` và `tcsetpgrp()` lần lượt lấy và thay đổi process group của terminal:

```c
#include <unistd.h>
pid_t tcgetpgrp(int fd);
int   tcsetpgrp(int fd, pid_t pgid);
```

`tcgetpgrp()` trả về PGID của foreground process group của terminal được tham chiếu bởi `fd`.

---

## 34.6 Signal SIGHUP

Khi controlling process mất kết nối terminal, kernel gửi `SIGHUP` để thông báo sự kiện này. Điều này thường xảy ra khi:
- Terminal disconnect được phát hiện (mất tín hiệu trên đường modem).
- Terminal window bị đóng.

Default action của `SIGHUP` là kết thúc process.

### 34.6.1 Xử Lý SIGHUP của Shell

Shell thường là controlling process cho terminal. Khi nhận `SIGHUP`, trước khi kết thúc, shell gửi `SIGHUP` đến mỗi process group nó đã tạo, khiến chúng kết thúc theo mặc định.

Lệnh `nohup(1)` làm cho lệnh miễn nhiễm với `SIGHUP`. Lệnh `disown` của bash xóa job khỏi danh sách job của shell.

### 34.6.2 SIGHUP và Kết Thúc Controlling Process

Nếu delivery `SIGHUP` khiến controlling process kết thúc, `SIGHUP` được gửi đến tất cả thành viên của foreground process group của controlling terminal. Điều này là hệ quả của việc kết thúc controlling process, không phải hành vi riêng của `SIGHUP`.

---

## 34.7 Job Control

Job control lần đầu xuất hiện khoảng năm 1980 trong C shell trên BSD. Job control cho phép người dùng shell đồng thời thực thi nhiều lệnh (job), một trong foreground và các lệnh khác trong background.

### 34.7.1 Sử Dụng Job Control Trong Shell

```bash
$ grep -r SIGHUP /usr/src/linux >x &    # Chạy trong background
[1] 18932
$ sleep 60 &
[2] 18934
$ jobs                                   # Liệt kê background job
[1]- Running  grep -r SIGHUP /usr/src/linux >x &
[2]+ Running  sleep 60 &
$ fg %1                                  # Đưa job vào foreground
```

Khi job đang chạy ở foreground, ta có thể suspend bằng ký tự terminal suspend (Control-Z), gửi `SIGTSTP` đến foreground process group. Sau đó dùng `fg` để resume ở foreground hoặc `bg` để resume ở background (shell gửi `SIGCONT`).

**Kiểm soát terminal I/O**:
- Chỉ process trong foreground job mới có thể đọc từ controlling terminal. Background job cố đọc nhận `SIGTTIN` (default: stop job).
- Nếu terminal TOSTOP flag được đặt, background job cố ghi nhận `SIGTTOU` (default: stop job).

### 34.7.2 Triển Khai Job Control

Job control yêu cầu:
1. Các job-control signal: `SIGTSTP`, `SIGSTOP`, `SIGCONT`, `SIGTTOU`, `SIGTTIN`.
2. Terminal driver hỗ trợ tạo job-control signal.
3. Shell hỗ trợ job control với các lệnh di chuyển job.

`SIGCONT` là ngoại lệ với quy tắc credentials thông thường: kernel cho phép process gửi `SIGCONT` đến bất kỳ process nào trong cùng session.

**Đặc biệt về SIGTTIN và SIGTTOU**:
- `SIGTTIN` không gửi nếu process đang block hoặc ignore signal; thay vào đó `read()` thất bại với `EIO`.
- Dù TOSTOP flag được đặt, `SIGTTOU` không gửi nếu process đang block hoặc ignore signal; `write()` được cho phép.

### 34.7.3 Xử Lý Job-Control Signal

Hầu hết ứng dụng không cần xử lý đặc biệt job-control signal. Ngoại lệ là các chương trình xử lý màn hình (như `vi`, `less`) cần:
- Khi nhận `SIGTSTP`: reset terminal về canonical mode, sau đó thực sự stop bằng cách raise `SIGTSTP` với default action.
- Khi resume qua `SIGCONT`: restore cài đặt terminal, kiểm tra kích thước cửa sổ, vẽ lại màn hình.

**Cách xử lý đúng SIGTSTP**:
1. Handler reset disposition của `SIGTSTP` về `SIG_DFL`.
2. Handler raise `SIGTSTP`.
3. Unblock `SIGTSTP` để pending signal thực hiện default action (process suspend).
4. Sau khi resume qua `SIGCONT`, execution của handler tiếp tục.
5. Reblock `SIGTSTP` và reestablish handler.

**Nguyên tắc chung**: Thiết lập handler cho job-control signal (`SIGTSTP`, `SIGTTIN`, `SIGTTOU`) và terminal signal (`SIGINT`, `SIGQUIT`, `SIGHUP`) chỉ nếu chúng không đang bị ignore (để tương thích với non-job-control shell).

### 34.7.4 Orphaned Process Group và SIGHUP

**Orphaned process group**: Process group mà "parent của mọi thành viên hoặc là thành viên của group đó hoặc không phải thành viên của session của group." Nói cách khác, process group không phải là orphaned nếu ít nhất một thành viên có parent trong cùng session nhưng trong process group khác.

Orphaned process group quan trọng vì không có process bên ngoài group có thể monitor các stopped process và gửi `SIGCONT` cho chúng — dẫn đến chúng có thể bị stuck mãi mãi.

**Giải pháp SUSv3**: Khi process group có stopped member trở nên orphaned, tất cả thành viên được gửi `SIGHUP` rồi `SIGCONT` để thông báo và đảm bảo chúng resume.

**Orphaned process group và job-control signal**: Thay vì gửi `SIGTTIN` hoặc `SIGTTOU` (sẽ gây stop vĩnh viễn), kernel làm cho `read()` hoặc `write()` thất bại với `EIO`. `SIGTSTP`, `SIGTTIN`, và `SIGTTOU` bị loại bỏ nếu chúng sẽ stop thành viên của orphaned process group.

---

## 34.8 Tóm Tắt

Session và process group (còn gọi là job) tạo thành phân cấp hai cấp: session là tập hợp process group, process group là tập hợp process. Session leader tạo session bằng `setsid()`, process group leader tạo group bằng `setpgid()`.

Session và process group được định nghĩa để hỗ trợ shell job control. Shell là session leader và controlling process cho terminal nó đang chạy. Mỗi job được tạo như một process group riêng biệt.

Terminal driver duy trì record của foreground process group và gửi job-control signal đến đó. SIGHUP được gửi đến controlling process khi terminal disconnect xảy ra.

Orphaned process group có thành viên bị stopped nhận `SIGHUP` rồi `SIGCONT` để tránh chúng bị stuck mãi mãi.

---

## 34.9 Bài Tập

- **34-1.** Nếu parent làm mình miễn nhiễm với `SIGUSR1` rồi dùng `killpg()` để gửi signal đến children, vấn đề nào có thể xảy ra với shell pipeline?
- **34-2.** Viết chương trình xác minh parent có thể thay đổi PGID của child trước khi child thực hiện `exec()`, nhưng không được sau đó.
- **34-3.** Viết chương trình xác minh `setsid()` từ process group leader thất bại.
- **34-4.** Xác minh nếu controlling process không kết thúc do nhận `SIGHUP`, kernel không gửi `SIGHUP` đến foreground process group.
- **34-5.** Nếu code unblock `SIGTSTP` được chuyển lên đầu handler, race condition nào có thể xảy ra?
- **34-6.** Viết chương trình xác minh khi process trong orphaned process group cố `read()` từ controlling terminal, `read()` thất bại với `EIO`.
- **34-7.** Xác minh `SIGTTIN`, `SIGTTOU`, hoặc `SIGTSTP` gửi đến orphaned process group bị loại bỏ nếu default action là stop, nhưng được delivery nếu handler được thiết lập.
