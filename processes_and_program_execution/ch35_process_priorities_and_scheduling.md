## Chương 35
# Ưu Tiên và Lên Lịch Process (Process Priorities and Scheduling)

Chương này thảo luận về các system call và thuộc tính process xác định khi nào và process nào được truy cập CPU. Chúng ta bắt đầu với nice value, sau đó là POSIX realtime scheduling API, và kết thúc với CPU affinity mask.

---

## 35.1 Ưu Tiên Process (Nice Value)

Mô hình mặc định để lên lịch process cho CPU là **round-robin time-sharing**. Theo mô hình này, mỗi process lần lượt được phép dùng CPU trong khoảng thời gian ngắn gọi là **time slice**.

Một thuộc tính process, **nice value**, cho phép process ảnh hưởng gián tiếp đến thuật toán scheduling của kernel. Mỗi process có nice value trong khoảng **–20 (ưu tiên cao) đến +19 (ưu tiên thấp)**, mặc định là 0. Trong các triển khai UNIX truyền thống, chỉ process có đặc quyền mới có thể đặt giá trị âm (ưu tiên cao). Process không có đặc quyền chỉ có thể giảm ưu tiên (tăng nice value).

Nice value kế thừa bởi child tạo qua `fork()` và được bảo toàn qua `exec()`.

### Lấy và thay đổi ưu tiên

```c
#include <sys/resource.h>
int getpriority(int which, id_t who);
           Returns (possibly negative) nice value of specified process on success,
                                                                   or –1 on error
int setpriority(int which, id_t who, int prio);
                                             Returns 0 on success, or –1 on error
```

Đối số `which` và `who` xác định process(es):
- **`PRIO_PROCESS`**: Process có process ID là `who` (0 = caller).
- **`PRIO_PGRP`**: Tất cả thành viên của process group có ID là `who` (0 = caller's group).
- **`PRIO_USER`**: Tất cả process có real user ID là `who` (0 = caller's real UID).

`getpriority()` có thể hợp lệ trả về `-1`, nên phải đặt `errno = 0` trước khi gọi và kiểm tra `errno` sau đó.

Thay đổi nice value qua `setpriority()` bị giới hạn bởi quyền: process không có đặc quyền chỉ có thể hạ ưu tiên của mình; process có đặc quyền (`CAP_SYS_NICE`) có thể thay đổi ưu tiên của bất kỳ process nào.

Từ kernel 2.6.12, `RLIMIT_NICE` resource limit cho phép process không có đặc quyền tăng nice value: process có thể đặt nice value tối đa là `20 – rlim_cur`.

---

## 35.2 Tổng Quan về Realtime Process Scheduling

Ứng dụng realtime có yêu cầu khắt khe hơn với scheduler:
- Phải cung cấp guaranteed maximum response time cho input bên ngoài.
- Process ưu tiên cao phải có khả năng giữ độc quyền CPU cho đến khi hoàn thành.
- Phải kiểm soát chính xác thứ tự scheduling của các process.

SUSv3 quy định POSIX realtime scheduling API với hai realtime scheduling policy: **`SCHED_RR`** và **`SCHED_FIFO`**. Process hoạt động dưới một trong hai policy này luôn có ưu tiên hơn process được scheduling bằng standard round-robin time-sharing policy (`SCHED_OTHER`).

Linux cung cấp 99 mức ưu tiên realtime, số từ 1 (thấp nhất) đến 99 (cao nhất).

### 35.2.1 Policy SCHED_RR

Dưới **SCHED_RR (round-robin)**, các process có cùng ưu tiên được thực thi theo kiểu round-robin time-sharing. Một process nhận time slice cố định mỗi lần dùng CPU. Process mất quyền truy cập CPU khi:
- Hết time slice → được đặt cuối queue của mức ưu tiên.
- Tự nguyện relinquish CPU → đặt cuối queue.
- Kết thúc.
- Bị preempt bởi process ưu tiên cao hơn → giữ đầu queue khi process ưu tiên cao hơn kết thúc.

### 35.2.2 Policy SCHED_FIFO

**SCHED_FIFO (first-in, first-out)** tương tự `SCHED_RR` nhưng **không có time slice**. Process `SCHED_FIFO` giữ CPU cho đến khi:
- Tự nguyện relinquish CPU.
- Kết thúc.
- Bị preempt bởi process ưu tiên cao hơn.

### 35.2.3 Policy SCHED_BATCH và SCHED_IDLE

Hai policy phi chuẩn:
- **`SCHED_BATCH`** (Linux 2.6.16): Tương tự `SCHED_OTHER` nhưng dành cho batch execution.
- **`SCHED_IDLE`** (Linux 2.6.23): Tương tự `SCHED_OTHER` nhưng ưu tiên thấp hơn cả nice value +19.

---

## 35.3 Realtime Process Scheduling API

### 35.3.1 Dải Ưu Tiên Realtime

```c
#include <sched.h>
int sched_get_priority_min(int policy);
int sched_get_priority_max(int policy);
             Both return nonnegative integer priority on success, or –1 on error
```

Trên Linux, cả hai hàm này trả về lần lượt 1 và 99 cho cả `SCHED_RR` và `SCHED_FIFO`. Nên chỉ định priority tương đối với giá trị trả về của các hàm này thay vì hard-code.

### 35.3.2 Thay Đổi và Lấy Policy và Priority

```c
#include <sched.h>
int sched_setscheduler(pid_t pid, int policy, const struct sched_param *param);
                                            Returns 0 on success, or –1 on error
```

`param` trỏ đến structure `sched_param` với trường `sched_priority`. Với `SCHED_RR` và `SCHED_FIFO`, priority phải trong dải `[1, 99]`; với các policy khác, priority phải là 0.

**Bảng 35-1:** Linux scheduling policy

| Policy | Mô tả | SUSv3 |
|--------|-------|-------|
| `SCHED_FIFO` | Realtime first-in first-out | ● |
| `SCHED_RR` | Realtime round-robin | ● |
| `SCHED_OTHER` | Standard round-robin time-sharing | ● |
| `SCHED_BATCH` | Tương tự `SCHED_OTHER` cho batch (từ Linux 2.6.16) | |
| `SCHED_IDLE` | Ưu tiên thấp hơn nice +19 (từ Linux 2.6.23) | |

Scheduling policy và priority kế thừa bởi child qua `fork()`, và được bảo toàn qua `exec()`.

```c
#include <sched.h>
int sched_setparam(pid_t pid, const struct sched_param *param);
                                            Returns 0 on success, or –1 on error
int sched_getscheduler(pid_t pid);
                                       Returns scheduling policy, or –1 on error
int sched_getparam(pid_t pid, struct sched_param *param);
                                            Returns 0 on success, or –1 on error
```

**Đặc quyền**: Từ kernel 2.6.12, process không có đặc quyền cũng có thể thay đổi scheduling nếu có `RLIMIT_RTPRIO` resource limit nonzero. Chỉ process có đặc quyền (`CAP_SYS_NICE`) mới có thể thay đổi tùy ý.

**Ngăn process realtime khóa hệ thống**: Khi phát triển ứng dụng dùng `SCHED_RR`/`SCHED_FIFO`, cần đặt resource limit (`RLIMIT_CPU`, `RLIMIT_RTTIME`) hoặc dùng alarm timer để tránh process bị lỗi chiếm CPU mãi mãi.

**`SCHED_RESET_ON_FORK`** (Linux 2.6.32): Flag này có thể OR với policy để đảm bảo child process không kế thừa realtime policy hay negative nice value.

### 35.3.3 Nhường CPU

```c
#include <sched.h>
int sched_yield(void);
                                             Returns 0 on success, or –1 on error
```

Nếu có process runnable khác cùng mức ưu tiên, process gọi được đặt cuối queue và process đầu queue được lên lịch. Nếu không có, process tiếp tục dùng CPU.

### 35.3.4 Time Slice của SCHED_RR

```c
#include <sched.h>
int sched_rr_get_interval(pid_t pid, struct timespec *tp);
                                             Returns 0 on success, or –1 on error
```

Trả về độ dài time slice của process `SCHED_RR`. Trên kernel 2.6 gần đây, time slice là 0.1 giây.

---

## 35.4 CPU Affinity

Trên hệ thống đa processor, khi process được lên lịch lại, nó không nhất thiết chạy trên CPU nó vừa chạy. Kernel Linux cố gắng đảm bảo **soft CPU affinity** — khi có thể, process được lên lịch lại trên cùng CPU.

Đôi khi cần đặt **hard CPU affinity** để giới hạn process chỉ chạy trên CPU(s) cụ thể:
- Tránh chi phí invalidation dữ liệu trong cache.
- Tối ưu khi nhiều thread cùng truy cập một dữ liệu.
- Dành riêng CPU cho ứng dụng time-critical.

```c
#define _GNU_SOURCE
#include <sched.h>
int sched_setaffinity(pid_t pid, size_t len, cpu_set_t *set);
int sched_getaffinity(pid_t pid, size_t len, cpu_set_t *set);
                                         Both return 0 on success, or –1 on error
```

Thao tác CPU set bằng các macro:

```c
void CPU_ZERO(cpu_set_t *set);     /* Khởi tạo set rỗng */
void CPU_SET(int cpu, cpu_set_t *set);   /* Thêm CPU vào set */
void CPU_CLR(int cpu, cpu_set_t *set);   /* Xóa CPU khỏi set */
int  CPU_ISSET(int cpu, cpu_set_t *set); /* Kiểm tra CPU trong set */
```

CPU trong set được đánh số bắt đầu từ 0. `CPU_SETSIZE` = 1024.

Ví dụ: giới hạn process chỉ chạy trên CPU 1, 2, 3 (của hệ thống 4 processor):

```c
cpu_set_t set;
CPU_ZERO(&set);
CPU_SET(1, &set);
CPU_SET(2, &set);
CPU_SET(3, &set);
sched_setaffinity(pid, CPU_SETSIZE, &set);
```

Child process kế thừa CPU affinity mask của parent, và mask này được bảo toàn qua `exec()`.

---

## 35.5 Tóm Tắt

Thuật toán scheduling kernel mặc định dùng round-robin time-sharing policy. Nice value trong khoảng –20 đến +19 ảnh hưởng đến mức CPU process nhận được. Process ưu tiên thấp không hoàn toàn bị starved.

Linux triển khai POSIX realtime scheduling extensions với hai policy: `SCHED_RR` và `SCHED_FIFO`. Process dưới hai policy này luôn có ưu tiên hơn nonrealtime process. Process `SCHED_FIFO` giữ CPU cho đến khi kết thúc hoặc nhường. Process `SCHED_RR` tương tự nhưng chia sẻ CPU theo round-robin giữa các process cùng mức ưu tiên.

CPU affinity mask giới hạn process chỉ chạy trên CPU(s) cụ thể trên hệ thống multiprocessor.

---

## 35.6 Bài Tập

- **35-1.** Triển khai lệnh `nice(1)`.
- **35-2.** Viết set-user-ID-root program là tương tự realtime của `nice(1)`.
- **35-3.** Viết chương trình đặt mình dưới `SCHED_FIFO` rồi tạo child process. Cả hai thực thi hàm tiêu thụ tối đa 3 giây CPU time. Sau mỗi giây CPU time, gọi `sched_yield()` để nhường CPU cho process kia. Kết quả phải cho thấy hai process xen kẽ nhau mỗi giây.
- **35-4.** Viết chương trình dùng `sched_setaffinity()` để chứng minh ảnh hưởng lên performance khi hai process trao đổi dữ liệu qua pipe chạy trên cùng CPU vs. CPU khác nhau.
