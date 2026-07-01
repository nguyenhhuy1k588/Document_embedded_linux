## Chương 39
# CAPABILITIES

Chương này mô tả cơ chế capabilities của Linux, cơ chế này chia nhỏ mô hình đặc quyền UNIX truyền thống (tất cả hoặc không có gì) thành các capabilities riêng lẻ có thể được bật hoặc tắt độc lập. Việc sử dụng capabilities cho phép một chương trình thực hiện một số thao tác đặc quyền, đồng thời ngăn nó thực hiện các thao tác khác.

# **39.1 Lý do ra đời của Capabilities**

Mô hình đặc quyền UNIX truyền thống chia các process thành hai loại: những process có effective user ID bằng 0 (superuser), vốn bỏ qua mọi kiểm tra đặc quyền, và tất cả các process còn lại, vốn phải chịu kiểm tra đặc quyền dựa trên user ID và group ID của chúng.

Tính phân chia thô này của mô hình trên là một vấn đề. Nếu chúng ta muốn cho phép một process thực hiện một thao tác nào đó chỉ được phép với superuser — ví dụ như thay đổi thời gian hệ thống — thì chúng ta phải chạy process đó với effective user ID bằng 0. (Nếu một người dùng không có đặc quyền cần thực hiện các thao tác đó, điều này thường được thực hiện thông qua một chương trình set-user-ID-root.) Tuy nhiên, điều này cấp cho process quyền thực hiện hàng loạt các thao tác khác — ví dụ như bỏ qua mọi kiểm tra quyền khi truy cập file — từ đó mở ra nguy cơ cho nhiều lỗ hổng bảo mật nếu chương trình hoạt động theo những cách không mong đợi (điều này có thể xảy ra do những tình huống không lường trước, hoặc do bị khai thác cố ý bởi người dùng có ý đồ xấu). Cách truyền thống để giải quyết vấn đề này đã được trình bày trong Chương [38]: chúng ta loại bỏ các đặc quyền effective (tức là thay đổi từ effective user ID bằng 0, trong khi vẫn duy trì 0 trong saved set-user-ID) và chỉ tạm thời lấy lại chúng khi cần.

Cơ chế capabilities của Linux tinh chỉnh cách xử lý vấn đề này. Thay vì sử dụng một đặc quyền duy nhất (tức là effective user ID bằng 0) khi thực hiện kiểm tra bảo mật trong kernel, đặc quyền superuser được chia thành các đơn vị riêng biệt, gọi là capabilities. Mỗi thao tác đặc quyền được gắn với một capability cụ thể, và một process chỉ có thể thực hiện thao tác đó nếu nó có capability tương ứng (bất kể effective user ID của nó là gì). Nói cách khác, ở mọi nơi trong cuốn sách này khi chúng ta nói về một process đặc quyền trên Linux, thực ra chúng ta đang nói về một process có capability liên quan để thực hiện một thao tác cụ thể.

Hầu hết thời gian, cơ chế capabilities của Linux là vô hình với chúng ta. Lý do là khi một ứng dụng không biết về capabilities giả định effective user ID bằng 0, kernel cấp cho process đó toàn bộ tập hợp capabilities.

Việc triển khai capabilities của Linux dựa trên tiêu chuẩn nháp POSIX 1003.1e (http://wt.tuxomania.net/publications/posix.1e/). Nỗ lực chuẩn hóa này bị đình trệ vào cuối những năm 1990 trước khi hoàn thành, nhưng nhiều triển khai capabilities vẫn dựa trên tiêu chuẩn nháp đó. (Một số capabilities được liệt kê trong Bảng 39-1 được định nghĩa trong tiêu chuẩn nháp POSIX.1e, nhưng nhiều capabilities khác là phần mở rộng của Linux.)

> Cơ chế capabilities được cung cấp trong một vài triển khai UNIX khác, chẳng hạn như Solaris 10 và các phiên bản Trusted Solaris trước đó của Sun, Trusted Irix của SGI, và như một phần của dự án TrustedBSD cho FreeBSD ([Watson, 2000]). Các cơ chế tương tự tồn tại trong một số hệ điều hành khác; ví dụ, cơ chế đặc quyền trong hệ thống VMS của Digital.

## **39.2 Các Linux Capabilities**

Bảng 39-1 liệt kê các Linux capabilities và cung cấp hướng dẫn ngắn gọn (và không đầy đủ) về các thao tác mà chúng áp dụng.

### **39.3 Capabilities của Process và File**

Mỗi process có ba tập hợp capabilities liên quan — gọi là permitted, effective và inheritable — có thể chứa không hoặc nhiều capabilities được liệt kê trong Bảng 39-1. Mỗi file cũng có thể có ba tập hợp capabilities liên quan, với cùng tên gọi. (Vì lý do sẽ trở nên rõ ràng sau, tập hợp effective capability của file thực ra chỉ là một bit duy nhất được bật hoặc tắt.) Chúng ta sẽ đi vào chi tiết của từng tập hợp capabilities này trong các phần tiếp theo.

### **39.3.1 Capabilities của Process**

Đối với mỗi process, kernel duy trì ba tập hợp capabilities (được triển khai dưới dạng bit mask) trong đó không hoặc nhiều capabilities được chỉ định trong Bảng 39-1 được bật. Ba tập hợp đó như sau:

- **Permitted:** Đây là các capabilities mà một process có thể sử dụng. Tập hợp permitted là superset giới hạn cho các capabilities có thể được thêm vào tập hợp effective và inheritable. Nếu một process loại bỏ một capability khỏi tập hợp permitted của nó, nó không bao giờ có thể lấy lại capability đó (trừ khi nó exec một chương trình một lần nữa cấp lại capability đó).

- **Effective:** Đây là các capabilities mà kernel sử dụng để thực hiện kiểm tra đặc quyền cho process. Miễn là nó duy trì một capability trong tập hợp permitted của mình, một process có thể tạm thời vô hiệu hóa capability bằng cách loại bỏ nó khỏi tập hợp effective, và sau đó khôi phục lại vào tập hợp đó.

- **Inheritable:** Đây là các capabilities có thể được chuyển sang tập hợp permitted khi một chương trình được exec bởi process này.

Chúng ta có thể xem biểu diễn hexadecimal của ba tập hợp capabilities cho bất kỳ process nào trong ba trường `CapInh`, `CapPrm` và `CapEff` trong file `/proc/PID/status` dành riêng cho Linux.

> Chương trình `getpcap` (là một phần của gói libcap được mô tả trong Mục [39.7]) có thể được sử dụng để hiển thị các capabilities của một process ở định dạng dễ đọc hơn.

Một process con được tạo ra thông qua `fork()` sẽ kế thừa các bản sao tập hợp capabilities của process cha. Chúng ta mô tả cách xử lý các tập hợp capabilities trong quá trình `exec()` trong Mục [39.5].

> Trên thực tế, capabilities là thuộc tính theo từng thread có thể được điều chỉnh độc lập cho mỗi thread trong một process. Capabilities của một thread cụ thể trong một process đa thread được hiển thị trong file `/proc/PID/task/TID/status`. File `/proc/PID/status` hiển thị capabilities của thread chính.

> Trước kernel 2.6.25, Linux biểu diễn các tập hợp capabilities bằng 32 bit. Việc bổ sung thêm capabilities trong kernel 2.6.25 đòi hỏi phải chuyển sang tập hợp 64 bit.

### **39.3.2 Capabilities của File**

Nếu một file có các tập hợp capabilities liên quan, thì các tập hợp này được sử dụng để xác định capabilities được cấp cho một process nếu nó exec file đó. Có ba tập hợp capabilities của file:

- **Permitted:** Đây là tập hợp các capabilities có thể được thêm vào tập hợp permitted của process trong quá trình `exec()`, bất kể các capabilities hiện có của process.

- **Effective:** Đây chỉ là một bit duy nhất. Nếu nó được bật, thì trong quá trình `exec()`, các capabilities được bật trong tập hợp permitted mới của process cũng được bật trong tập hợp effective mới của process. Nếu bit effective của file bị tắt, thì sau một `exec()`, tập hợp effective mới của process ban đầu sẽ rỗng.

- **Inheritable:** Tập hợp này được AND với tập hợp inheritable của process để xác định một tập hợp các capabilities sẽ được bật trong tập hợp permitted của process sau một `exec()`.

Mục 39.5 cung cấp chi tiết về cách các capabilities của file được sử dụng trong quá trình `exec()`.

Các capabilities permitted và inheritable của file trước đây được gọi là forced và allowed. Những thuật ngữ đó giờ đã lỗi thời, nhưng chúng vẫn cung cấp thông tin hữu ích. Các capabilities permitted của file là những capabilities được buộc vào tập hợp permitted của process trong quá trình `exec()`, bất kể các capabilities hiện có của process. Các capabilities inheritable của file là những capabilities mà file cho phép vào tập hợp permitted của process trong quá trình `exec()`, nếu những capabilities đó cũng được bật trong tập hợp inheritable capabilities của process.

Các capabilities liên quan đến một file được lưu trữ trong một security extended attribute (Mục 16.1) có tên là `security.capability`. Capability `CAP_SETFCAP` là bắt buộc để cập nhật extended attribute này.

**Bảng 39-1:** Các thao tác được phép bởi từng Linux capability

| Capability           | Cho phép process                                                                                                                                                                                                                                                                                                                                                                                                 |
|----------------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `CAP_AUDIT_CONTROL`    | (Từ Linux 2.6.11) Bật và tắt ghi nhật ký audit của kernel; thay đổi các quy tắc lọc cho audit; truy xuất trạng thái audit và các quy tắc lọc                                                                                                                                                                                                                                                                 |
| `CAP_AUDIT_WRITE`      | (Từ Linux 2.6.11) Ghi bản ghi vào nhật ký audit của kernel                                                                                                                                                                                                                                                                                                                                                      |
| `CAP_CHOWN`            | Thay đổi user ID (owner) của file hoặc thay đổi group ID của file thành một group mà process không là thành viên (`chown()`)                                                                                                                                                                                                                                                                                   |
| `CAP_DAC_OVERRIDE`     | Bỏ qua kiểm tra quyền đọc, ghi và thực thi file (DAC là viết tắt của discretionary access control); đọc nội dung các symbolic link `cwd`, `exe` và `root` trong `/proc/PID`                                                                                                                                                                                                                                            |
| `CAP_DAC_READ_SEARCH`  | Bỏ qua kiểm tra quyền đọc file và kiểm tra quyền đọc và thực thi (tìm kiếm) thư mục                                                                                                                                                                                                                                                                                                                    |
| `CAP_FOWNER`           | Nhìn chung bỏ qua kiểm tra quyền trên các thao tác thường yêu cầu file-system user ID của process khớp với user ID của file (`chmod()`, `utime()`); đặt cờ i-node trên các file tùy ý; đặt và sửa đổi ACL trên các file tùy ý; bỏ qua ảnh hưởng của sticky bit thư mục khi xóa file (`unlink()`, `rmdir()`, `rename()`); chỉ định cờ `O_NOATIME` cho các file tùy ý trong `open()` và `fcntl(F_SETFL)` |
| `CAP_FSETID`           | Sửa đổi file mà không có kernel tắt các bit set-user-ID và set-group-ID (`write()`, `truncate()`); bật bit set-group-ID cho file có group ID không khớp với file-system group ID hoặc supplementary group IDs của process (`chmod()`)                                                                                                                                                                         |
| `CAP_IPC_LOCK`         | Ghi đè các hạn chế khóa bộ nhớ (`mlock()`, `mlockall()`, `shmctl(SHM_LOCK)`, `shmctl(SHM_UNLOCK)`); sử dụng cờ `SHM_HUGETLB` của `shmget()` và cờ `MAP_HUGETLB` của `mmap()`.                                                                                                                                                                                                                                           |
| `CAP_IPC_OWNER`        | Bỏ qua kiểm tra quyền cho các thao tác trên các đối tượng System V IPC                                                                                                                                                                                                                                                                                                                                    |
| `CAP_KILL`             | Bỏ qua kiểm tra quyền khi gửi signal (`kill()`, `sigqueue()`)                                                                                                                                                                                                                                                                                                                                                  |
| `CAP_LEASE`            | (Từ Linux 2.4) Thiết lập lease trên các file tùy ý (`fcntl(F_SETLEASE)`)                                                                                                                                                                                                                                                                                                                                          |
| `CAP_LINUX_IMMUTABLE`  | Đặt cờ i-node append và immutable                                                                                                                                                                                                                                                                                                                                                                              |
| `CAP_MAC_ADMIN`        | (Từ Linux 2.6.25) Cấu hình hoặc thực hiện thay đổi trạng thái cho mandatory access control (MAC) (được triển khai bởi một số Linux security module)                                                                                                                                                                                                                                                            |
| `CAP_MAC_OVERRIDE`     | (Từ Linux 2.6.25) Ghi đè MAC (được triển khai bởi một số Linux security module)                                                                                                                                                                                                                                                                                                                                  |
| `CAP_MKNOD`            | (Từ Linux 2.4) Sử dụng `mknod()` để tạo thiết bị                                                                                                                                                                                                                                                                                                                                                                    |
| `CAP_NET_ADMIN`        | Thực hiện các thao tác liên quan đến mạng (ví dụ: đặt các tùy chọn socket đặc quyền, bật multicasting, cấu hình giao diện mạng và sửa đổi bảng định tuyến)                                                                                                                                                                                                                                                    |
| `CAP_NET_BIND_SERVICE` | Liên kết với các cổng socket đặc quyền                                                                                                                                                                                                                                                                                                                                                                         |
| `CAP_NET_BROADCAST`    | (Không dùng) Thực hiện phát sóng socket và lắng nghe multicast                                                                                                                                                                                                                                                                                                                                                 |
| `CAP_NET_RAW`          | Sử dụng raw socket và packet socket                                                                                                                                                                                                                                                                                                                                                                             |
| `CAP_SETGID`           | Thực hiện các thay đổi tùy ý với group ID của process (`setgid()`, `setegid()`, `setregid()`, `setresgid()`, `setfsgid()`, `setgroups()`, `initgroups()`); giả mạo group ID khi truyền thông tin xác thực qua UNIX domain socket (`SCM_CREDENTIALS`)                                                                                                                                                          |
| `CAP_SETFCAP`          | (Từ Linux 2.6.24) Đặt file capabilities                                                                                                                                                                                                                                                                                                                                                                         |

**Bảng 39-1:** Các thao tác được phép bởi từng Linux capability (tiếp theo)

| Capability         | Cho phép process                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
|--------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `CAP_SETPCAP`        | Nếu file capabilities không được hỗ trợ, cấp và thu hồi capabilities trong tập hợp permitted của process cho hoặc từ bất kỳ process nào khác (bao gồm bản thân); nếu file capabilities được hỗ trợ, thêm bất kỳ capability nào trong capability bounding set của process vào tập hợp inheritable của nó, loại bỏ capabilities khỏi bounding set và thay đổi các cờ securebits                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `CAP_SETUID`         | Thực hiện các thay đổi tùy ý với user ID của process (`setuid()`, `seteuid()`, `setreuid()`, `setresuid()`, `setfsuid()`); giả mạo user ID khi truyền thông tin xác thực qua UNIX domain socket (`SCM_CREDENTIALS`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| `CAP_SYS_ADMIN`      | Vượt giới hạn `/proc/sys/fs/file-max` trong các system call mở file (ví dụ: `open()`, `shm_open()`, `pipe()`, `socket()`, `accept()`, `exec()`, `acct()`, `epoll_create()`); thực hiện các thao tác quản trị hệ thống khác nhau, bao gồm `quotactl()` (kiểm soát quota đĩa), `mount()` và `umount()`, `swapon()` và `swapoff()`, `pivot_root()`, `sethostname()` và `setdomainname()`; thực hiện các thao tác `syslog(2)` khác nhau; ghi đè giới hạn tài nguyên `RLIMIT_NPROC` (`fork()`); gọi `lookup_dcookie()`; đặt trusted và security extended attribute; thực hiện các thao tác `IPC_SET` và `IPC_RMID` trên các đối tượng System V IPC tùy ý; giả mạo process ID khi truyền thông tin xác thực qua UNIX domain socket (`SCM_CREDENTIALS`); sử dụng `ioprio_set()` để gán scheduling class `IOPRIO_CLASS_RT`; sử dụng `ioctl()` `TIOCCONS`; sử dụng cờ `CLONE_NEWNS` với `clone()` và `unshare()`; thực hiện các thao tác `keyctl()` `KEYCTL_CHOWN` và `KEYCTL_SETPERM`; quản lý thiết bị `random(4)`; các thao tác dành riêng cho thiết bị khác |
| `CAP_SYS_BOOT`       | Sử dụng `reboot()` để khởi động lại hệ thống; gọi `kexec_load()`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| `CAP_SYS_CHROOT`     | Sử dụng `chroot()` để đặt thư mục gốc của process                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| `CAP_SYS_MODULE`     | Nạp và gỡ bỏ kernel module (`init_module()`, `delete_module()`, `create_module()`)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| `CAP_SYS_NICE`       | Tăng giá trị nice (`nice()`, `setpriority()`); thay đổi giá trị nice cho các process tùy ý (`setpriority()`); đặt các chính sách lập lịch realtime `SCHED_RR` và `SCHED_FIFO` cho calling process; đặt lại cờ `SCHED_RESET_ON_FORK`; đặt chính sách và ưu tiên lập lịch cho các process tùy ý (`sched_setscheduler()`, `sched_setparam()`); đặt I/O scheduling class và ưu tiên cho các process tùy ý (`ioprio_set()`); đặt CPU affinity cho các process tùy ý (`sched_setaffinity()`); sử dụng `migrate_pages()` để di chuyển các process tùy ý và cho phép các process được di chuyển đến các node tùy ý; áp dụng `move_pages()` cho các process tùy ý; sử dụng cờ `MPOL_MF_MOVE_ALL` với `mbind()` và `move_pages()`                                                                                                                                                                                                                                                                 |
| `CAP_SYS_PACCT`      | Sử dụng `acct()` để bật hoặc tắt process accounting                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| `CAP_SYS_PTRACE`     | Theo dõi các process tùy ý bằng `ptrace()`; truy cập `/proc/PID/environ` cho các process tùy ý; áp dụng `get_robust_list()` cho các process tùy ý                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `CAP_SYS_RAWIO`      | Thực hiện các thao tác trên cổng I/O bằng `iopl()` và `ioperm()`; truy cập `/proc/kcore`; mở `/dev/mem` và `/dev/kmem`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `CAP_SYS_RESOURCE`   | Sử dụng không gian dự trữ trên file system; thực hiện các lệnh gọi `ioctl()` kiểm soát journaling ext3; ghi đè giới hạn quota đĩa; tăng hard resource limit (`setrlimit()`); ghi đè giới hạn tài nguyên `RLIMIT_NPROC` (`fork()`); tăng giới hạn `msg_qbytes` cho message queue System V vượt giới hạn trong `/proc/sys/kernel/msgmnb`; bỏ qua các giới hạn POSIX message queue được định nghĩa bởi các file dưới `/proc/sys/fs/mqueue`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `CAP_SYS_TIME`       | Sửa đổi đồng hồ hệ thống (`settimeofday()`, `stime()`, `adjtime()`, `adjtimex()`); đặt hardware clock                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| `CAP_SYS_TTY_CONFIG` | Thực hiện virtual hangup của terminal hoặc pseudoterminal bằng `vhangup()`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |

### **39.3.3 Mục đích của tập hợp Permitted và Effective Capabilities của Process**

Tập hợp capabilities permitted của process xác định các capabilities mà một process có thể sử dụng. Tập hợp capabilities effective của process xác định các capabilities hiện đang có hiệu lực cho process — tức là tập hợp các capabilities mà kernel sử dụng khi kiểm tra xem process có đủ đặc quyền để thực hiện một thao tác cụ thể hay không.

Tập hợp capabilities permitted áp đặt giới hạn trên cho tập hợp effective. Một process chỉ có thể tăng một capability trong tập hợp effective của nó nếu capability đó nằm trong tập hợp permitted. (Các thuật ngữ "thêm vào" và "đặt" đôi khi được dùng đồng nghĩa với "tăng". Thao tác ngược lại là "loại bỏ", hay đồng nghĩa, "xóa".)

> Mối quan hệ giữa tập hợp effective và permitted capabilities tương tự như mối quan hệ giữa effective user ID và saved set-user-ID cho một chương trình setuser-ID-root. Việc loại bỏ một capability khỏi tập hợp effective tương tự như việc tạm thời loại bỏ effective user ID bằng 0, trong khi vẫn duy trì 0 trong saved set-user-ID. Việc loại bỏ một capability khỏi cả tập hợp effective lẫn permitted tương tự như việc vĩnh viễn loại bỏ đặc quyền superuser bằng cách đặt cả effective user ID lẫn saved set-user ID thành các giá trị khác không.

### **39.3.4 Mục đích của tập hợp Permitted và Effective Capabilities của File**

Tập hợp capabilities permitted của file cung cấp một cơ chế mà một file thực thi có thể cấp capabilities cho một process. Nó chỉ định một nhóm capabilities sẽ được gán vào tập hợp permitted capabilities của process trong quá trình `exec()`.

Tập hợp capabilities effective của file là một cờ (bit) duy nhất được bật hoặc tắt. Để hiểu tại sao tập hợp này chỉ gồm một bit duy nhất, chúng ta cần xem xét hai trường hợp xảy ra khi một chương trình được exec:

- Chương trình có thể là **capability-dumb**, nghĩa là nó không biết về capabilities (tức là được thiết kế như một chương trình set-user-ID-root truyền thống). Chương trình như vậy không biết rằng nó cần tăng capabilities trong tập hợp effective của mình để có thể thực hiện các thao tác đặc quyền. Đối với các chương trình như vậy, một `exec()` nên có tác dụng là tất cả các capabilities permitted mới của process cũng được tự động gán vào tập hợp effective của nó. Kết quả này đạt được bằng cách bật bit effective của file.

- Chương trình có thể là **capability-aware**, nghĩa là nó được thiết kế có tính đến framework capabilities, và nó sẽ thực hiện các system call phù hợp (được thảo luận sau) để tăng và giảm capabilities trong tập hợp effective của nó. Đối với các chương trình như vậy, các cân nhắc về đặc quyền tối thiểu có nghĩa là sau một `exec()`, tất cả các capabilities ban đầu sẽ bị tắt trong tập hợp capabilities effective của process. Kết quả này đạt được bằng cách tắt bit capabilities effective của file.

### **39.3.5 Mục đích của tập hợp Inheritable của Process và File**

Nhìn thoáng qua, việc sử dụng tập hợp permitted và effective cho process và file có vẻ là một framework đủ dùng cho hệ thống capabilities. Tuy nhiên, có một số tình huống mà chúng không đủ. Ví dụ, điều gì sẽ xảy ra nếu một process đang thực hiện `exec()` muốn bảo toàn một số capabilities hiện tại của nó qua `exec()`? Có vẻ như việc triển khai capabilities có thể cung cấp tính năng này đơn giản bằng cách bảo toàn các capabilities permitted của process qua một `exec()`. Tuy nhiên, cách tiếp cận này sẽ không xử lý được các trường hợp sau:

- Việc thực hiện `exec()` có thể yêu cầu một số đặc quyền nhất định (ví dụ: `CAP_DAC_OVERRIDE`) mà chúng ta không muốn bảo toàn qua `exec()`.

- Giả sử chúng ta đã loại bỏ một số capabilities permitted mà chúng ta không muốn bảo toàn qua `exec()`, nhưng sau đó `exec()` thất bại. Trong trường hợp này, chương trình có thể cần một số capabilities permitted mà nó đã (không thể thu hồi) loại bỏ.

Vì những lý do này, các capabilities permitted của process không được bảo toàn qua một `exec()`. Thay vào đó, một tập hợp capabilities khác được giới thiệu: tập hợp inheritable. Tập hợp inheritable cung cấp một cơ chế mà một process có thể bảo toàn một số capabilities của nó qua một `exec()`.

Tập hợp capabilities inheritable của process chỉ định một nhóm capabilities có thể được gán vào tập hợp permitted capabilities của process trong quá trình `exec()`. Tập hợp inheritable của file tương ứng được AND với tập hợp capabilities kế thừa của process để xác định các capabilities thực sự được thêm vào tập hợp permitted capabilities của process trong quá trình `exec()`.

> Có một lý do triết học hơn để không đơn giản bảo toàn tập hợp capabilities permitted của process qua một `exec()`. Ý tưởng của hệ thống capabilities là tất cả các đặc quyền được cấp cho một process đều được cấp hoặc kiểm soát bởi file mà process exec. Mặc dù tập hợp inheritable của process chỉ định các capabilities được truyền qua một `exec()`, những capabilities này bị AND bởi tập hợp inheritable của file.

# **39.3.6 Gán và Xem File Capabilities từ Shell**

Các lệnh `setcap(8)` và `getcap(8)`, có trong gói libcap được mô tả trong Mục [39.7], có thể được sử dụng để thao tác các tập hợp capabilities của file. Chúng ta minh họa việc sử dụng các lệnh này với một ví dụ ngắn sử dụng chương trình `date(1)` tiêu chuẩn. (Chương trình này là một ví dụ về ứng dụng capability-dumb theo định nghĩa trong Mục [39.3.4].) Khi chạy với đặc quyền, `date(1)` có thể được sử dụng để thay đổi thời gian hệ thống. Chương trình `date` không phải là set-user-ID-root, vì vậy thông thường cách duy nhất để chạy nó với đặc quyền là trở thành superuser.

Chúng ta bắt đầu bằng cách hiển thị thời gian hệ thống hiện tại, rồi thử thay đổi thời gian với tư cách là người dùng không có đặc quyền:

```
$ date
Tue Dec 28 15:54:08 CET 2010
$ date -s '2018-02-01 21:39'
date: cannot set date: Operation not permitted
Thu Feb 1 21:39:00 CET 2018
```

Ở trên, chúng ta thấy rằng lệnh `date` không thể thay đổi thời gian hệ thống, nhưng vẫn hiển thị đối số của nó theo định dạng chuẩn.

Tiếp theo, chúng ta trở thành superuser, điều này cho phép chúng ta thay đổi thời gian hệ thống thành công:

```
$ sudo date -s '2018-02-01 21:39'
root's password:
Thu Feb 1 21:39:00 CET 2018
$ date
Thu Feb 1 21:39:02 CET 2018
```

Bây giờ chúng ta tạo một bản sao của chương trình `date` và gán cho nó capability mà nó cần:

```
$ whereis -b date Find location of date binary
date: /bin/date
$ cp /bin/date .
$ sudo setcap "cap_sys_time=pe" date
root's password:
$ getcap date
date = cap_sys_time+ep
```

Lệnh `setcap` ở trên gán capability `CAP_SYS_TIME` vào tập hợp permitted (p) và effective (e) capabilities của file thực thi. Sau đó chúng ta sử dụng lệnh `getcap` để xác minh các capabilities được gán cho file. (Cú pháp được sử dụng bởi `setcap` và `getcap` để biểu diễn các tập hợp capabilities được mô tả trong trang man `cap_from_text(3)` được cung cấp trong gói libcap.)

Các file capabilities của bản sao chương trình `date` của chúng ta cho phép chương trình này được sử dụng bởi người dùng không có đặc quyền để đặt thời gian hệ thống:

```
$ ./date -s '2010-12-28 15:55'
Tue Dec 28 15:55:00 CET 2010
$ date
Tue Dec 28 15:55:02 CET 2010
```

### **39.4 Triển khai Capabilities Hiện đại**

Một triển khai capabilities đầy đủ yêu cầu những điều sau:

- Đối với mỗi thao tác đặc quyền, kernel nên kiểm tra xem process có capability liên quan hay không, thay vì kiểm tra effective (hoặc file system) user ID bằng 0.
- Kernel phải cung cấp các system call cho phép capabilities của process được truy xuất và sửa đổi.
- Kernel phải hỗ trợ khái niệm gắn capabilities vào một file thực thi, để process có được các capabilities liên quan khi file đó được exec. Điều này tương tự như set-user-ID bit, nhưng cho phép chỉ định độc lập tất cả các capabilities trên file thực thi. Ngoài ra, hệ thống phải cung cấp một tập hợp các giao diện lập trình và lệnh để đặt và xem các capabilities được gắn vào file thực thi.

Cho đến và bao gồm kernel 2.6.23, Linux chỉ đáp ứng hai yêu cầu đầu tiên trong số này. Từ kernel 2.6.24, có thể gắn capabilities vào một file. Nhiều tính năng khác đã được thêm vào kernel 2.6.25 và 2.6.26 để hoàn thiện việc triển khai capabilities.

Đối với phần lớn thảo luận về capabilities của chúng ta, chúng ta sẽ tập trung vào triển khai hiện đại. Trong Mục [39.10], chúng ta xem xét cách triển khai khác biệt trước khi file capabilities được giới thiệu. Hơn nữa, file capabilities là một thành phần kernel tùy chọn trong các kernel hiện đại, nhưng đối với phần chính của thảo luận, chúng ta sẽ giả định rằng thành phần này được bật. Sau đó, chúng ta sẽ mô tả các sự khác biệt xảy ra nếu file capabilities không được bật. (Về nhiều khía cạnh, hành vi tương tự như Linux trong các kernel trước 2.6.24, nơi file capabilities chưa được triển khai.)

Trong các phần tiếp theo, chúng ta sẽ đi vào chi tiết hơn về việc triển khai capabilities của Linux.

### **39.5 Biến đổi Capabilities của Process trong quá trình exec()**

Trong quá trình `exec()`, kernel đặt các capabilities mới cho process dựa trên các capabilities hiện tại của process và các tập hợp capabilities của file đang được thực thi. Kernel tính toán các capabilities mới của process bằng cách sử dụng các quy tắc sau:

```
P'(permitted) = (P(inheritable) & F(inheritable)) | (F(permitted) & cap_bset)
P'(effective) = F(effective) ? P'(permitted) : 0
P'(inheritable) = P(inheritable)
```

Trong các quy tắc trên, `P` ký hiệu giá trị của một tập hợp capabilities trước `exec()`, `P'` ký hiệu giá trị của một tập hợp capabilities sau `exec()`, và `F` ký hiệu một tập hợp capabilities của file. Định danh `cap_bset` ký hiệu giá trị của capability bounding set. Lưu ý rằng `exec()` để nguyên tập hợp capabilities inheritable của process không thay đổi.

### **39.5.1 Capability Bounding Set**

Capability bounding set là một cơ chế bảo mật được sử dụng để giới hạn các capabilities mà một process có thể đạt được trong quá trình `exec()`. Tập hợp này được sử dụng như sau:

- Trong quá trình `exec()`, capability bounding set được AND với các capabilities permitted của file để xác định các capabilities permitted sẽ được cấp cho chương trình mới. Nói cách khác, tập hợp capabilities permitted của một file thực thi không thể cấp một capability permitted cho một process nếu capability đó không nằm trong bounding set.

- Capability bounding set là superset giới hạn cho các capabilities có thể được thêm vào tập hợp inheritable của process. Điều này có nghĩa là, trừ khi capability nằm trong bounding set, một process không thể thêm một trong các capabilities permitted của nó vào tập hợp inheritable của nó và sau đó — thông qua quy tắc biến đổi capability đầu tiên được mô tả ở trên — có capability đó được bảo toàn trong tập hợp permitted của nó khi nó exec một file có capability trong tập hợp inheritable của nó.

Capability bounding set là thuộc tính theo từng process được kế thừa bởi một process con được tạo thông qua `fork()`, và được bảo toàn qua một `exec()`. Trên kernel hỗ trợ file capabilities, `init` (tổ tiên của tất cả các process) bắt đầu với một capability bounding set chứa tất cả các capabilities.

Nếu một process có capability `CAP_SETPCAP`, thì nó có thể (không thể đảo ngược) xóa capabilities khỏi bounding set của nó bằng cách sử dụng thao tác `prctl()` `PR_CAPBSET_DROP`. (Việc loại bỏ một capability khỏi bounding set không ảnh hưởng đến các tập hợp capabilities permitted, effective và inheritable của process.) Một process có thể xác định xem một capability có trong bounding set của nó hay không bằng cách sử dụng thao tác `prctl()` `PR_CAPBSET_READ`.

> Chính xác hơn, capability bounding set là thuộc tính theo từng thread. Bắt đầu từ Linux 2.6.26, thuộc tính này được hiển thị dưới dạng trường `CapBnd` trong file `/proc/PID/task/TID/status` dành riêng cho Linux. File `/proc/PID/status` hiển thị bounding set của thread chính của process.

### **39.5.2 Bảo toàn ngữ nghĩa của root**

Để bảo toàn ngữ nghĩa truyền thống cho người dùng root (tức là root có tất cả các đặc quyền) khi thực thi một file, bất kỳ tập hợp capabilities nào liên quan đến file đều bị bỏ qua. Thay vào đó, cho mục đích của thuật toán được hiển thị trong Mục [39.5], các tập hợp capabilities của file được định nghĩa theo cách danh nghĩa như sau trong quá trình `exec()`:

- Nếu một chương trình set-user-ID-root đang được exec, hoặc real hay effective user ID của process gọi `exec()` là 0, thì các tập hợp inheritable và permitted của file được định nghĩa là tất cả các bit bằng 1.
- Nếu một chương trình set-user-ID-root đang được exec, hoặc effective user ID của process gọi `exec()` là 0, thì bit effective của file được định nghĩa là được đặt.

Giả sử chúng ta đang exec một chương trình set-user-ID-root, các định nghĩa danh nghĩa về tập hợp capabilities của file này có nghĩa là việc tính toán các tập hợp capabilities permitted và effective mới của process trong Mục [39.5] được đơn giản hóa thành như sau:

```
P'(permitted) = P(inheritable) | cap_bset
P'(effective) = P'(permitted)
```

## **39.6 Ảnh hưởng đến Capabilities của Process khi Thay đổi User ID**

Để duy trì tính tương thích với các ý nghĩa truyền thống cho các chuyển đổi giữa user ID 0 và user ID khác không, kernel thực hiện các thao tác sau khi thay đổi user ID của process (sử dụng `setuid()`, v.v.):

1. Nếu real user ID, effective user ID hoặc saved set-user-ID trước đây có giá trị 0 và, kết quả từ các thay đổi với user ID, cả ba ID này đều có giá trị khác không, thì các tập hợp capabilities permitted và effective sẽ bị xóa (tức là tất cả các capabilities bị loại bỏ vĩnh viễn).

2. Nếu effective user ID được thay đổi từ 0 sang giá trị khác không, thì tập hợp capabilities effective sẽ bị xóa (tức là các capabilities effective bị loại bỏ, nhưng những capabilities trong tập hợp permitted có thể được tăng trở lại).

3. Nếu effective user ID được thay đổi từ giá trị khác không sang 0, thì tập hợp capabilities permitted được sao chép vào tập hợp capabilities effective (tức là tất cả các capabilities permitted trở thành effective).

4. Nếu file-system user ID được thay đổi từ 0 sang giá trị khác không, thì các capabilities liên quan đến file sau đây bị xóa khỏi tập hợp capabilities effective: `CAP_CHOWN`, `CAP_DAC_OVERRIDE`, `CAP_DAC_READ_SEARCH`, `CAP_FOWNER`, `CAP_FSETID`, `CAP_LINUX_IMMUTABLE` (từ Linux 2.6.30), `CAP_MAC_OVERRIDE` và `CAP_MKNOD` (từ Linux 2.6.30). Ngược lại, nếu file-system user ID được thay đổi từ giá trị khác không sang 0, thì bất kỳ capabilities nào trong số này được bật trong tập hợp permitted đều được bật trong tập hợp effective. Các thao tác này được thực hiện để duy trì ngữ nghĩa truyền thống cho các thao tác với file-system user ID dành riêng cho Linux.

# **39.7 Thay đổi Capabilities của Process theo Chương trình**

Một process có thể tăng hoặc giảm capabilities từ các tập hợp capabilities của nó bằng cách sử dụng system call `capset()` hoặc, tốt hơn là sử dụng libcap API, mà chúng ta mô tả bên dưới. Các thay đổi đối với capabilities của process phải tuân theo các quy tắc sau:

1. Nếu process không có capability `CAP_SETPCAP` trong tập hợp effective của nó, thì tập hợp inheritable mới phải là tập con của sự kết hợp của tập hợp inheritable và permitted hiện có.

2. Tập hợp inheritable mới phải là tập con của sự kết hợp của tập hợp inheritable hiện có và capability bounding set.

3. Tập hợp permitted mới phải là tập con của tập hợp permitted hiện có. Nói cách khác, một process không thể tự cấp cho mình các capabilities permitted mà nó không có. Nói cách khác, một capability đã bị loại bỏ khỏi tập hợp permitted không thể được lấy lại.

4. Tập hợp effective mới chỉ được phép chứa các capabilities cũng có trong tập hợp permitted mới.

### **libcap API**

Cho đến thời điểm này, chúng ta đã cố tình không hiển thị prototype của system call `capset()`, hoặc đối tác của nó `capget()`, vốn truy xuất capabilities của một process. Điều này là bởi vì việc sử dụng các system call này nên được tránh. Thay vào đó, nên sử dụng các hàm trong thư viện libcap. Các hàm này cung cấp một giao diện tuân theo tiêu chuẩn nháp POSIX 1003.1e đã bị rút lại, cùng với một số phần mở rộng Linux.

Vì lý do không gian, chúng ta không mô tả libcap API một cách chi tiết. Nhìn chung, các chương trình sử dụng các hàm này thường thực hiện các bước sau:

1. Sử dụng hàm `cap_get_proc()` để lấy một bản sao các tập hợp capabilities hiện tại của process từ kernel và đặt nó vào một cấu trúc mà hàm cấp phát trong user space. (Ngoài ra, chúng ta có thể sử dụng hàm `cap_init()` để tạo một cấu trúc tập hợp capabilities mới, rỗng.) Trong libcap API, kiểu dữ liệu `cap_t` là một pointer được sử dụng để tham chiếu đến các cấu trúc đó.

2. Sử dụng hàm `cap_set_flag()` để cập nhật cấu trúc user-space để tăng (`CAP_SET`) và giảm (`CAP_CLEAR`) các capabilities từ các tập hợp permitted, effective và inheritable được lưu trong cấu trúc user-space được lấy ở bước trước.

3. Sử dụng hàm `cap_set_proc()` để chuyển cấu trúc user-space trở lại kernel để thay đổi capabilities của process.

4. Sử dụng hàm `cap_free()` để giải phóng cấu trúc được cấp phát bởi libcap API trong bước đầu tiên.

Vào thời điểm viết bài này, công việc đang được tiến hành với libcap-ng, một libcap API mới và được cải tiến. Chi tiết có thể được tìm thấy tại http://freshmeat.net/projects/libcap-ng.

#### **Chương trình ví dụ**

Trong Listing 8-2, trang 164, chúng ta đã trình bày một chương trình xác thực tên người dùng và mật khẩu đối với cơ sở dữ liệu mật khẩu tiêu chuẩn. Chúng ta đã lưu ý rằng chương trình yêu cầu đặc quyền để đọc file mật khẩu shadow, vốn được bảo vệ để ngăn người dùng khác ngoài root hoặc thành viên nhóm shadow đọc. Cách truyền thống để cung cấp cho chương trình này các đặc quyền cần thiết là chạy nó dưới đăng nhập root hoặc làm nó thành chương trình set-user-ID-root. Chúng ta bây giờ trình bày một phiên bản sửa đổi của chương trình này sử dụng capabilities và libcap API.

Để đọc file mật khẩu shadow với tư cách là người dùng thông thường, chúng ta cần bỏ qua các kiểm tra quyền file tiêu chuẩn. Nhìn qua các capabilities được liệt kê trong Bảng 39-1, chúng ta thấy rằng capability phù hợp là `CAP_DAC_READ_SEARCH`. Phiên bản sửa đổi của chương trình xác thực mật khẩu được hiển thị trong [Listing 39-1]. Chương trình này sử dụng libcap API để tăng `CAP_DAC_READ_SEARCH` trong tập hợp capabilities effective của nó ngay trước khi truy cập file mật khẩu shadow, và sau đó loại bỏ capability đó ngay sau khi truy cập này. Để một người dùng không có đặc quyền có thể sử dụng chương trình, chúng ta phải đặt capability này trong tập hợp capabilities permitted của file, như được hiển thị trong phiên làm việc shell sau:

```
$ sudo setcap "cap_dac_read_search=p" check_password_caps
root's password:
$ getcap check_password_caps
check_password_caps = cap_dac_read_search+p
$ ./check_password_caps
Username: mtk
Password:
Successfully authenticated: UID=1000
```

**Listing 39-1:** Một chương trình capability-aware xác thực người dùng

```
––––––––––––––––––––––––––––––––––––––––––––––––– cap/check_password_caps.c
#define _BSD_SOURCE /* Get getpass() declaration from <unistd.h> */
#define _XOPEN_SOURCE /* Get crypt() declaration from <unistd.h> */
#include <sys/capability.h>
#include <unistd.h>
#include <limits.h>
#include <pwd.h>
#include <shadow.h>
#include "tlpi_hdr.h"
/* Change setting of capability in caller's effective capabilities */
static int
modifyCap(int capability, int setting)
```

```
{
 cap_t caps;
 cap_value_t capList[1];
 /* Retrieve caller's current capabilities */
 caps = cap_get_proc();
 if (caps == NULL)
 return -1;
 /* Change setting of 'capability' in the effective set of 'caps'. The
 third argument, 1, is the number of items in the array 'capList'. */
 capList[0] = capability;
 if (cap_set_flag(caps, CAP_EFFECTIVE, 1, capList, setting) == -1) {
 cap_free(caps);
 return -1;
 }
 /* Push modified capability sets back to kernel, to change
 caller's capabilities */
 if (cap_set_proc(caps) == -1) {
 cap_free(caps);
 return -1;
 }
 /* Free the structure that was allocated by libcap */
 if (cap_free(caps) == -1)
 return -1;
 return 0;
}
static int /* Raise capability in caller's effective set */
raiseCap(int capability)
{
 return modifyCap(capability, CAP_SET);
}
/* An analogous dropCap() (unneeded in this program), could be
 defined as: modifyCap(capability, CAP_CLEAR); */
static int /* Drop all capabilities from all sets */
dropAllCaps(void)
{
 cap_t empty;
 int s;
 empty = cap_init();
 if (empty == NULL)
 return -1;
 s = cap_set_proc(empty);
```

```
 if (cap_free(empty) == -1)
 return -1;
 return s;
}
int
main(int argc, char *argv[])
{
 char *username, *password, *encrypted, *p;
 struct passwd *pwd;
 struct spwd *spwd;
 Boolean authOk;
 size_t len;
 long lnmax;
 lnmax = sysconf(_SC_LOGIN_NAME_MAX);
 if (lnmax == -1) /* If limit is indeterminate */
 lnmax = 256; /* make a guess */
 username = malloc(lnmax);
 if (username == NULL)
 errExit("malloc");
 printf("Username: ");
 fflush(stdout);
 if (fgets(username, lnmax, stdin) == NULL)
 exit(EXIT_FAILURE); /* Exit on EOF */
 len = strlen(username);
 if (username[len - 1] == '\n')
 username[len - 1] = '\0'; /* Remove trailing '\n' */
 pwd = getpwnam(username);
 if (pwd == NULL)
 fatal("couldn't get password record");
 /* Only raise CAP_DAC_READ_SEARCH for as long as we need it */
 if (raiseCap(CAP_DAC_READ_SEARCH) == -1)
 fatal("raiseCap() failed");
 spwd = getspnam(username);
 if (spwd == NULL && errno == EACCES)
 fatal("no permission to read shadow password file");
 /* At this point, we won't need any more capabilities,
 so drop all capabilities from all sets */
 if (dropAllCaps() == -1)
 fatal("dropAllCaps() failed");
 if (spwd != NULL) /* If there is a shadow password record */
 pwd->pw_passwd = spwd->sp_pwdp; /* Use the shadow password */
```

```
 password = getpass("Password: ");
 /* Encrypt password and erase cleartext version immediately */
 encrypted = crypt(password, pwd->pw_passwd);
 for (p = password; *p != '\0'; )
 *p++ = '\0';
 if (encrypted == NULL)
 errExit("crypt");
 authOk = strcmp(encrypted, pwd->pw_passwd) == 0;
 if (!authOk) {
 printf("Incorrect password\n");
 exit(EXIT_FAILURE);
 }
 printf("Successfully authenticated: UID=%ld\n", (long) pwd->pw_uid);
 /* Now do authenticated work... */
 exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––– cap/check_password_caps.c
```

# **39.8 Tạo môi trường chỉ dùng Capabilities**

Trong các trang trước, chúng ta đã mô tả các cách khác nhau mà một process với user ID 0 (root) được xử lý đặc biệt liên quan đến capabilities:

- Khi một process có một hoặc nhiều user ID bằng 0 đặt tất cả user ID của nó thành các giá trị khác không, các tập hợp capabilities permitted và effective của nó sẽ bị xóa. (Xem Mục [39.6].)

- Khi một process có effective user ID bằng 0 thay đổi user ID đó sang giá trị khác không, nó mất các effective capabilities. Khi thực hiện thay đổi ngược lại, tập hợp capabilities permitted được sao chép vào tập hợp effective. Một quy trình tương tự được áp dụng cho một tập hợp con capabilities khi file-system user ID của process được chuyển đổi giữa 0 và các giá trị khác không. (Xem Mục [39.6].)

- Nếu một process có real hoặc effective user ID là root exec một chương trình, hoặc bất kỳ process nào exec một chương trình set-user-ID-root, thì các tập hợp inheritable và permitted của file được định nghĩa theo cách danh nghĩa là tất cả các bit bằng 1. Nếu effective user ID của process là 0, hoặc nó đang exec một chương trình set-user-ID-root, thì bit effective của file được định nghĩa theo cách danh nghĩa là 1. (Xem Mục [39.5.2].) Trong các trường hợp thông thường (tức là cả real và effective user ID đều là root, hoặc một chương trình set-user-ID-root đang được exec), điều này có nghĩa là process nhận được tất cả các capabilities trong tập hợp permitted và effective của nó.

Trong một hệ thống hoàn toàn dựa trên capabilities, kernel sẽ không cần thực hiện bất kỳ xử lý đặc biệt nào cho root. Sẽ không có chương trình set-user-ID-root, và file capabilities sẽ được sử dụng để cấp đúng các capabilities tối thiểu mà một chương trình yêu cầu.

Vì các ứng dụng hiện tại chưa được thiết kế để sử dụng cơ sở hạ tầng file-capabilities, kernel phải duy trì cách xử lý truyền thống các process có user ID 0. Tuy nhiên, chúng ta có thể muốn một ứng dụng chạy trong môi trường hoàn toàn dựa trên capabilities trong đó root không nhận được bất kỳ xử lý đặc biệt nào được mô tả ở trên. Bắt đầu với kernel 2.6.26, và nếu file capabilities được bật trong kernel, Linux cung cấp cơ chế securebits, kiểm soát một tập hợp các cờ theo từng process bật hoặc tắt mỗi trong ba xử lý đặc biệt cho root. (Để chính xác, các cờ securebits thực sự là thuộc tính theo từng thread.)

Cơ chế securebits kiểm soát các cờ được hiển thị trong [Bảng 39-2]. Các cờ tồn tại dưới dạng các cặp liên quan gồm một cờ cơ sở và một cờ khóa tương ứng. Mỗi cờ cơ sở kiểm soát một trong các xử lý đặc biệt cho root được mô tả ở trên. Việc đặt cờ khóa tương ứng là một thao tác một lần ngăn các thay đổi thêm vào cờ cơ sở liên quan — một khi được đặt, cờ khóa không thể bị bỏ đặt.

**Bảng 39-2: Các cờ securebits**

| Cờ                          | Ý nghĩa khi được đặt                                                                                                                                                                                                                               |
|-------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `SECBIT_KEEP_CAPS`              | Không loại bỏ các capabilities permitted khi một process có một hoặc nhiều user ID bằng 0 đặt tất cả user ID của nó thành các giá trị khác không. Cờ này chỉ có hiệu lực nếu `SECBIT_NO_SETUID_FIXUP` cũng không được đặt. Cờ này bị xóa khi `exec()`. |
| `SECBIT_NO_SETUID_FIXUP`        | Không thay đổi capabilities khi effective hoặc file-system user ID được chuyển đổi giữa 0 và các giá trị khác không.                                                                                                                               |
| `SECBIT_NOROOT`                 | Nếu một process có real hoặc effective user ID bằng 0 thực hiện `exec()`, hoặc nó exec một chương trình set-user-ID-root, đừng cấp capabilities cho nó (trừ khi file thực thi có file capabilities).                                                  |
| `SECBIT_KEEP_CAPS_LOCKED`       | Khóa `SECBIT_KEEP_CAPS`.                                                                                                                                                                                                                       |
| `SECBIT_NO_SETUID_FIXUP_LOCKED` | Khóa `SECBIT_NO_SETUID_FIXUP`.                                                                                                                                                                                                                 |
| `SECBIT_NOROOT_LOCKED`          | Khóa `SECBIT_NOROOT`.                                                                                                                                                                                                                          |

Các cài đặt cờ securebits được kế thừa trong một process con được tạo bởi `fork()`. Tất cả các cài đặt cờ được bảo toàn trong quá trình `exec()`, ngoại trừ `SECBIT_KEEP_CAPS`, vốn bị xóa để tương thích lịch sử với cài đặt `PR_SET_KEEPCAPS` của `prctl()`, được mô tả bên dưới.

Một process có thể truy xuất các cờ securebits bằng cách sử dụng thao tác `prctl()` `PR_GET_SECUREBITS`. Nếu một process có capability `CAP_SETPCAP`, nó có thể sửa đổi các cờ securebits bằng cách sử dụng các thao tác `prctl()` `PR_SET_SECUREBITS`. Một ứng dụng hoàn toàn dựa trên capabilities có thể vô hiệu hóa vĩnh viễn xử lý đặc biệt root cho calling process và tất cả các con cháu của nó bằng cách sử dụng lệnh gọi sau:

```
if (prctl(PR_SET_SECUREBITS,
 /* SECBIT_KEEP_CAPS off */
 SECBIT_NO_SETUID_FIXUP | SECBIT_NO_SETUID_FIXUP_LOCKED |
 SECBIT_NOROOT | SECBIT_NOROOT_LOCKED)
 == -1)
 errExit("prctl");
```

Sau lệnh gọi này, cách duy nhất để process này và các con cháu của nó có được capabilities là bằng cách thực thi các chương trình có file capabilities.

### **SECBIT\_KEEP\_CAPS và thao tác prctl() PR\_SET\_KEEPCAPS**

Cờ `SECBIT_KEEP_CAPS` ngăn các capabilities bị loại bỏ khi một process có một hoặc nhiều user ID với giá trị 0 đặt tất cả user ID của nó thành các giá trị khác không. Nói một cách đơn giản, `SECBIT_KEEP_CAPS` cung cấp một nửa chức năng được cung cấp bởi `SECBIT_NO_SETUID_FIXUP`. (Như đã lưu ý trong [Bảng 39-2], `SECBIT_KEEP_CAPS` chỉ có hiệu lực nếu `SECBIT_NO_SETUID_FIXUP` không được đặt.) Cờ này tồn tại để cung cấp một cờ securebits phản ánh thao tác `prctl()` `PR_SET_KEEPCAPS` cũ hơn, vốn kiểm soát cùng thuộc tính. (Một sự khác biệt giữa hai cơ chế là một process không cần capability `CAP_SETPCAP` để sử dụng thao tác `prctl()` `PR_SET_KEEPCAPS`.)

> Trước đó, chúng ta đã lưu ý rằng tất cả các cờ securebits được bảo toàn trong quá trình `exec()`, ngoại trừ `SECBIT_KEEP_CAPS`. Việc đặt bit `SECBIT_KEEP_CAPS` được tạo ra ngược với các cài đặt securebits khác để duy trì tính nhất quán với cách xử lý thuộc tính được đặt bởi thao tác `prctl()` `PR_SET_KEEPCAPS`.

Thao tác `prctl()` `PR_SET_KEEPCAPS` được thiết kế để sử dụng bởi các chương trình set-user-ID-root chạy trên các kernel cũ hơn không hỗ trợ file capabilities. Các chương trình như vậy vẫn có thể cải thiện bảo mật của chúng bằng cách loại bỏ và tăng capabilities theo chương trình khi cần (xem Mục [39.10]).

Tuy nhiên, ngay cả khi một chương trình set-user-ID-root như vậy loại bỏ tất cả các capabilities ngoại trừ những capabilities mà nó yêu cầu, nó vẫn duy trì hai đặc quyền quan trọng: khả năng truy cập các file thuộc sở hữu của root và khả năng lấy lại capabilities bằng cách exec một chương trình (Mục [39.5.2]). Cách duy nhất để vĩnh viễn loại bỏ các đặc quyền này là đặt tất cả user ID của process thành các giá trị khác không. Nhưng làm điều đó thường dẫn đến việc xóa các tập hợp capabilities permitted và effective (xem bốn điểm trong Mục [39.6] liên quan đến ảnh hưởng của các thay đổi user ID đối với capabilities). Điều này làm hỏng mục đích, là vĩnh viễn loại bỏ user ID 0, trong khi vẫn duy trì một số capabilities. Để cho phép khả năng này, thao tác `prctl()` `PR_SET_KEEPCAPS` có thể được sử dụng để đặt thuộc tính process ngăn tập hợp capabilities permitted bị xóa khi tất cả user ID được thay đổi thành giá trị khác không. (Tập hợp capabilities effective của process luôn bị xóa trong trường hợp này, bất kể cài đặt thuộc tính "keep capabilities".)

### **39.9 Khám phá các Capabilities mà Chương trình Yêu cầu**

Giả sử chúng ta có một chương trình không biết về capabilities và chỉ được cung cấp ở dạng nhị phân, hoặc chúng ta có một chương trình có mã nguồn quá lớn để chúng ta dễ dàng đọc để xác định các capabilities có thể được yêu cầu để chạy nó. Nếu chương trình yêu cầu đặc quyền, nhưng không nên là chương trình set-user-ID-root, thì làm thế nào chúng ta có thể xác định các capabilities permitted để gán cho file thực thi với `setcap(8)`? Có hai cách để trả lời câu hỏi này:

- Sử dụng `strace(1)` (Phụ lục A) để xem system call nào thất bại với lỗi `EPERM`, lỗi được sử dụng để chỉ ra sự thiếu hụt của một capability được yêu cầu. Bằng cách tham khảo trang man của system call hoặc mã nguồn kernel, chúng ta có thể suy ra capability nào được yêu cầu. Cách tiếp cận này không hoàn hảo, vì lỗi `EPERM` đôi khi có thể được tạo ra vì các lý do khác, một số trong số đó có thể không liên quan gì đến các yêu cầu capability của chương trình. Hơn nữa, các chương trình có thể hợp lệ thực hiện một system call yêu cầu đặc quyền, và sau đó thay đổi hành vi của chúng sau khi xác định rằng chúng không có đặc quyền cho một thao tác cụ thể. Đôi khi có thể khó phân biệt các "false positive" như vậy khi cố gắng xác định các capabilities mà một file thực thi thực sự cần.

- Sử dụng kernel probe để tạo đầu ra giám sát khi kernel được yêu cầu thực hiện kiểm tra capability. Một ví dụ về cách thực hiện điều này được cung cấp trong [Hallyn, 2007], một bài viết được viết bởi một trong các nhà phát triển file capabilities. Đối với mỗi yêu cầu kiểm tra một capability, probe được hiển thị trong bài viết ghi lại hàm kernel được gọi, capability được yêu cầu và tên của chương trình yêu cầu. Mặc dù cách tiếp cận này đòi hỏi nhiều công việc hơn so với việc sử dụng `strace(1)`, nó cũng có thể giúp chúng ta xác định chính xác hơn các capabilities mà một chương trình yêu cầu.

# **39.10 Kernel Cũ và Hệ thống Không có File Capabilities**

Trong mục này, chúng ta mô tả các sự khác biệt khác nhau trong việc triển khai capabilities trong các kernel cũ hơn. Chúng ta cũng mô tả các sự khác biệt xảy ra trên các kernel nơi file capabilities không được hỗ trợ. Có hai tình huống mà Linux không hỗ trợ file capabilities:

- Trước Linux 2.6.24, file capabilities chưa được triển khai.
- Từ Linux 2.6.24, file capabilities có thể bị tắt nếu kernel được build mà không có tùy chọn `CONFIG_SECURITY_FILE_CAPABILITIES`.

Mặc dù Linux đã giới thiệu capabilities và cho phép chúng được gắn vào các process bắt đầu từ kernel 2.2, việc triển khai file capabilities chỉ xuất hiện vài năm sau đó. Lý do mà file capabilities vẫn chưa được triển khai lâu như vậy là vấn đề chính sách, chứ không phải khó khăn kỹ thuật. (Extended attribute, được mô tả trong Chương 16, vốn được sử dụng để triển khai file capabilities, đã có sẵn từ kernel 2.6.) Quan điểm của các nhà phát triển kernel là việc yêu cầu quản trị viên hệ thống đặt và giám sát các tập hợp capabilities khác nhau — mà một số trong đó có hậu quả tinh tế nhưng sâu rộng — cho mỗi chương trình đặc quyền sẽ tạo ra một nhiệm vụ quản trị phức tạp không thể quản lý được. Ngược lại, quản trị viên hệ thống đã quen thuộc với mô hình đặc quyền UNIX hiện có, biết cách xử lý các chương trình set-user-ID một cách thận trọng, và có thể tìm các chương trình set-user-ID và set-group-ID trên hệ thống bằng cách sử dụng các lệnh `find` đơn giản. Tuy nhiên, các nhà phát triển file capabilities đã lập luận rằng file capabilities có thể được tổ chức để hoạt động được về mặt hành chính, và cuối cùng cung cấp đủ lập luận thuyết phục để file capabilities được tích hợp vào kernel.

### **Capability CAP\_SETPCAP**

Trên các kernel không hỗ trợ file capabilities (tức là bất kỳ kernel nào trước 2.6.24, và các kernel từ 2.6.24 trở đi với file capabilities bị tắt), ngữ nghĩa của capability `CAP_SETPCAP` là khác nhau. Theo các quy tắc tương tự như những quy tắc được mô tả trong Mục [39.7], một process có capability `CAP_SETPCAP` trong tập hợp effective của nó về lý thuyết có thể thay đổi các capabilities của các process khác với chính nó. Các thay đổi có thể được thực hiện với capabilities của một process khác, tất cả các thành viên của một process group được chỉ định, hoặc tất cả các process trên hệ thống ngoại trừ `init` và chính caller. Trường hợp cuối cùng loại trừ `init` vì nó cơ bản với hoạt động của hệ thống. Nó cũng loại trừ caller vì caller có thể đang cố gắng xóa capabilities khỏi mọi process khác trên hệ thống, và chúng ta không muốn xóa các capabilities khỏi calling process chính nó.

Tuy nhiên, việc thay đổi capabilities của các process khác chỉ là khả năng lý thuyết. Trên các kernel cũ hơn, và trên các kernel hiện đại nơi hỗ trợ file capabilities bị tắt, capability bounding set (được thảo luận tiếp theo) luôn che khuất capability `CAP_SETPCAP`.

### **Capability bounding set**

Từ Linux 2.6.25, capability bounding set là thuộc tính theo từng process. Tuy nhiên, trên các kernel cũ hơn, capability bounding set là thuộc tính toàn hệ thống ảnh hưởng đến tất cả các process trên hệ thống. Capability bounding set toàn hệ thống được khởi tạo để nó luôn che khuất `CAP_SETPCAP` (được mô tả ở trên).

> Trên các kernel sau 2.6.25, việc xóa capabilities khỏi per-process bounding set chỉ được hỗ trợ nếu file capabilities được bật trong kernel. Trong trường hợp đó, `init`, tổ tiên của tất cả các process, bắt đầu với bounding set chứa tất cả các capabilities, và một bản sao của bounding set đó được kế thừa bởi các process khác được tạo trên hệ thống. Nếu file capabilities bị tắt, thì, do sự khác biệt trong ngữ nghĩa của `CAP_SETPCAP` được mô tả ở trên, `init` bắt đầu với bounding set chứa tất cả các capabilities ngoại trừ `CAP_SETPCAP`.

Có một sự thay đổi nữa trong ngữ nghĩa của capability bounding set trong Linux 2.6.25. Như đã lưu ý trước đó ([Mục 39.5.1]), trên Linux 2.6.25 và sau đó, per-process capability bounding set hoạt động như một superset giới hạn cho các capabilities có thể được thêm vào tập hợp inheritable của process. Trong Linux 2.6.24 và trước đó, capability bounding set toàn hệ thống không có hiệu lực che khuất này. (Nó không cần thiết, vì các kernel này không hỗ trợ file capabilities.)

Capability bounding set toàn hệ thống có thể truy cập thông qua file `/proc/sys/kernel/cap-bound` dành riêng cho Linux. Một process phải có capability `CAP_SYS_MODULE` để có thể thay đổi nội dung của `cap-bound`. Tuy nhiên, chỉ process `init` mới có thể bật các bit trong mask này; các process đặc quyền khác chỉ có thể tắt các bit. Kết quả của những giới hạn này là trên một hệ thống nơi file capabilities không được hỗ trợ, chúng ta không bao giờ có thể cấp capability `CAP_SETPCAP` cho một process. Điều này hợp lý, vì capability đó có thể được sử dụng để phá vỡ toàn bộ hệ thống kiểm tra đặc quyền kernel. (Trong trường hợp khó xảy ra khi chúng ta muốn thay đổi giới hạn này, chúng ta phải nạp một kernel module thay đổi giá trị trong tập hợp, sửa đổi mã nguồn của chương trình `init`, hoặc thay đổi việc khởi tạo capability bounding set trong mã nguồn kernel và thực hiện rebuild kernel.)

> Một cách gây nhầm lẫn, mặc dù đó là bit mask, giá trị trong file `cap-bound` toàn hệ thống được hiển thị dưới dạng số thập phân có dấu. Ví dụ, giá trị ban đầu của file này là -257. Đây là diễn giải bù hai của bit mask với tất cả các bit ngoại trừ (1 << 8) được bật (tức là trong nhị phân, 11111111 11111111 11111110 11111111); `CAP_SETPCAP` có giá trị 8.

### **Sử dụng capabilities trong một chương trình trên hệ thống không có file capabilities**

Ngay cả trên một hệ thống không hỗ trợ file capabilities, chúng ta vẫn có thể sử dụng capabilities để cải thiện bảo mật của một chương trình. Chúng ta thực hiện điều này như sau:

1. Chạy chương trình trong một process có effective user ID bằng 0 (thường là chương trình set-user-ID-root). Một process như vậy được cấp tất cả các capabilities (ngoại trừ `CAP_SETPCAP`, như đã lưu ý trước đó) trong các tập hợp permitted và effective của nó.

2. Khi khởi động chương trình, sử dụng libcap API để loại bỏ tất cả các capabilities khỏi tập hợp effective, và loại bỏ tất cả các capabilities ngoại trừ những capabilities mà chúng ta có thể cần sau này khỏi tập hợp permitted.

3. Đặt cờ `SECBIT_KEEP_CAPS` (hoặc sử dụng thao tác `prctl()` `PR_SET_KEEPCAPS` để đạt được kết quả tương tự), để bước tiếp theo không loại bỏ capabilities.

4. Đặt tất cả user ID thành các giá trị khác không, để ngăn process truy cập các file thuộc sở hữu của root hoặc có được capabilities bằng cách thực hiện `exec()`.

Chúng ta có thể thay thế hai bước trước bằng một bước duy nhất đặt cờ `SECBIT_NOROOT`, nếu chúng ta muốn ngăn process lấy lại đặc quyền khi `exec()`, nhưng phải cho phép nó truy cập các file thuộc sở hữu của root. (Tất nhiên, việc cho phép truy cập vào các file thuộc sở hữu của root để lại nguy cơ về một số lỗ hổng bảo mật.)

5. Trong phần còn lại của thời gian tồn tại của chương trình, sử dụng libcap API để tăng và giảm các capabilities permitted còn lại khỏi tập hợp effective khi cần thiết để thực hiện các nhiệm vụ đặc quyền.

Một số ứng dụng được xây dựng cho các kernel Linux trước phiên bản 2.6.24 đã sử dụng cách tiếp cận này.

> Trong số các nhà phát triển kernel phản đối việc triển khai capabilities cho các file thực thi, một trong những ưu điểm được nhận thức của cách tiếp cận được mô tả trong văn bản chính là nhà phát triển ứng dụng biết các capabilities nào mà file thực thi yêu cầu. Ngược lại, quản trị viên hệ thống có thể không dễ dàng xác định thông tin này.

# **39.11 Tóm tắt**

Cơ chế capabilities của Linux chia các thao tác đặc quyền thành các danh mục riêng biệt, và cho phép một process được cấp một số capabilities, trong khi bị từ chối các capabilities khác. Cơ chế này đại diện cho một sự cải tiến so với cơ chế đặc quyền truyền thống tất cả hoặc không có gì, theo đó một process có đặc quyền để thực hiện tất cả các thao tác (user ID 0) hoặc không có đặc quyền (user ID khác không). Từ kernel 2.6.24, Linux hỗ trợ gắn capabilities vào các file, để một process có thể có được các capabilities được chọn bằng cách exec một chương trình.

### **39.12 Bài tập**

**39-1.** Sửa đổi chương trình trong [Listing 35-2] (sched\_set.c, trang 743) để sử dụng file capabilities, để nó có thể được sử dụng bởi người dùng không có đặc quyền.
