## Chương 45
# Giới Thiệu về System V IPC

System V IPC là nhãn dùng để chỉ ba cơ chế interprocess communication khác nhau:

- **Message queue**: Có thể dùng để truyền message giữa các process. Tương tự pipe nhưng có biên giới message được bảo toàn, và mỗi message bao gồm trường integer type cho phép chọn message theo type.
- **Semaphore**: Cho phép nhiều process đồng bộ hóa hành động. Semaphore là integer value được kernel duy trì và hiển thị với tất cả process có quyền phù hợp.
- **Shared memory**: Cho phép nhiều process chia sẻ cùng vùng bộ nhớ (cùng page frame được map vào virtual memory của nhiều process). Đây là một trong những phương thức IPC nhanh nhất.

---

## 45.1 Tổng Quan API

**Bảng 45-1:** Tóm tắt các giao diện lập trình cho System V IPC object

| Giao diện | Message queue | Semaphore | Shared memory |
|-----------|--------------|-----------|---------------|
| Header file | `<sys/msg.h>` | `<sys/sem.h>` | `<sys/shm.h>` |
| Cấu trúc dữ liệu liên quan | `msqid_ds` | `semid_ds` | `shmid_ds` |
| Tạo/mở object | `msgget()` | `semget()` | `shmget()` + `shmat()` |
| Đóng object | (không có) | (không có) | `shmdt()` |
| Thao tác điều khiển | `msgctl()` | `semctl()` | `shmctl()` |
| Thực hiện IPC | `msgsnd()`, `msgrcv()` | `semop()` | Truy cập vùng nhớ |

**Tạo và mở System V IPC object:**

Mỗi cơ chế System V IPC có lời gọi get liên quan (`msgget()`, `semget()`, `shmget()`), tương tự `open()` cho file. Với integer key (tương tự filename), lời gọi get hoặc:
- Tạo IPC object mới với key đã cho và trả về identifier duy nhất; hoặc
- Trả về identifier của IPC object hiện có với key đã cho.

IPC identifier tương tự file descriptor nhưng có khác biệt quan trọng: file descriptor là thuộc tính process, còn IPC identifier là thuộc tính của object và hiển thị trên toàn hệ thống. Tất cả process truy cập cùng object dùng cùng identifier.

```c
id = msgget(key, IPC_CREAT | S_IRUSR | S_IWUSR);
if (id == -1)  errExit("msgget");
```

Process umask không được áp dụng cho quyền của IPC object mới tạo.

**Xóa IPC object:**

Lời gọi `ctl` (`msgctl()`, `semctl()`, `shmctl()`) thực hiện nhiều thao tác điều khiển. `IPC_RMID` xóa object. Với message queue và semaphore, xóa là ngay lập tức và thông tin bị hủy bất kể có process nào đang dùng không. Với shared memory, segment chỉ bị xóa sau khi tất cả process detach nó qua `shmdt()`.

**Object persistence:** System V IPC object có **kernel persistence** — tồn tại cho đến khi bị xóa rõ ràng hoặc hệ thống tắt.

---

## 45.2 IPC Key

System V IPC key là integer value, kiểu `key_t`. Có ba cách tạo key:

1. **Chọn ngẫu nhiên** một giá trị integer (rủi ro xung đột với ứng dụng khác).

2. **Dùng `IPC_PRIVATE`** làm key khi tạo object — luôn tạo object mới với key duy nhất:
   ```c
   id = msgget(IPC_PRIVATE, S_IRUSR | S_IWUSR);
   ```
   Hữu ích trong ứng dụng đa process nơi parent tạo object trước `fork()`.

3. **Dùng `ftok()`** để tạo key từ pathname và giá trị proj:
   ```c
   #include <sys/ipc.h>
   key_t ftok(char *pathname, int proj);
                                     Returns integer key on success, or –1 on error
   ```
   `ftok()` tạo key từ i-node number của file (không phải tên file), minor device number, và 8 bit thấp của `proj`. File không nên bị xóa và tạo lại vì có thể có i-node number khác. `proj` cho phép tạo nhiều key khác nhau từ cùng file.

   ```c
   key_t key;
   key = ftok("/mydir/myfile", 'x');
   if (key == -1)  errExit("ftok");
   id = msgget(key, IPC_CREAT | S_IRUSR | S_IWUSR);
   ```

---

## 45.3 Cấu Trúc Dữ Liệu Liên Quan và Quyền Object

Kernel duy trì cấu trúc dữ liệu liên quan cho mỗi instance của System V IPC object. Tất cả ba cơ chế IPC đều có substructure `ipc_perm` chứa thông tin quyền:

```c
struct ipc_perm {
    key_t          __key; /* Key, như cung cấp cho lời gọi 'get' */
    uid_t          uid;   /* User ID của owner */
    gid_t          gid;   /* Group ID của owner */
    uid_t          cuid;  /* User ID của creator */
    gid_t          cgid;  /* Group ID của creator */
    unsigned short mode;  /* Quyền */
    unsigned short __seq; /* Sequence number */
};
```

Ban đầu, owner ID và creator ID có cùng giá trị từ effective ID của process gọi. Creator ID là immutable; owner ID có thể thay đổi qua `IPC_SET`. Trường `mode` giữ permission mask.

**Kiểm tra quyền:**
1. Process có đặc quyền (`CAP_IPC_OWNER`) → tất cả quyền được cấp.
2. Effective user ID khớp với owner hoặc creator user ID → quyền owner (user) được cấp.
3. Effective group ID hoặc supplementary group ID khớp với owner/creator group ID → quyền group được cấp.
4. Ngược lại → quyền other được cấp.

Chỉ quyền **read** và **write** có ý nghĩa với IPC object (không có execute). Để xóa hoặc thay đổi cấu trúc dữ liệu liên quan, cần đặc quyền (`CAP_SYS_ADMIN`) hoặc effective user ID khớp với owner/creator user ID.

---

## 45.4 IPC Identifier và Ứng Dụng Client-Server

Trong ứng dụng client-server, server thường tạo System V IPC object (dùng `IPC_CREAT`) còn client chỉ truy cập chúng. Nếu server bị crash và khởi động lại, server nên xóa IPC object cũ và tạo mới:

```c
while ((msqid = msgget(key, IPC_CREAT | IPC_EXCL | MQ_PERMS)) == -1) {
    if (errno == EEXIST) {
        msqid = msgget(key, 0);
        if (msgctl(msqid, IPC_RMID, NULL) == -1)
            errExit("msgget() failed to delete old queue");
    } else {
        errExit("msgget() failed");
    }
}
```

---

## 45.5 Thuật Toán của System V IPC get Call

Kernel duy trì structure `ipc_ids` cho mỗi cơ chế IPC, chứa mảng con trỏ (`entries`) đến cấu trúc dữ liệu liên quan.

Khi tạo IPC object mới, giá trị `seq` của `ipc_ids` được sao chép vào `xxx_perm.__seq` và tăng lên một. Identifier được tính bằng:

```
identifier = index + xxx_perm.__seq * SEQ_MULTIPLIER
```

Trong đó `SEQ_MULTIPLIER = 32,768` và `index` là chỉ mục trong mảng `entries`.

Ưu điểm: Ngay cả khi IPC object mới được tạo với cùng key, nó sẽ có identifier khác vì `seq` đã tăng lên, khiến client của server cũ nhận lỗi khi cố dùng identifier cũ.

---

## 45.6 Lệnh ipcs và ipcrm

`ipcs` là lệnh tương tự `ls` cho System V IPC:

```
$ ipcs
------ Shared Memory Segments --------
key        shmid  owner  perms  bytes  nattch  status
0x6d0731db 262147 mtk    600    8192   2
------ Semaphore Arrays --------
key        semid  owner    perms  nsems
0x6107c0b8 0      cecilia  660    6
------ Message Queues --------
key        msqid  owner    perms  used-bytes  messages
0x71075958 229376 cecilia  620    12          2
```

`ipcrm` xóa IPC object:

```bash
$ ipcrm -s 65538   # Xóa semaphore với identifier 65538
$ ipcrm -q id      # Xóa message queue
$ ipcrm -m id      # Xóa shared memory
```

---

## 45.7 Lấy Danh Sách Tất Cả IPC Object

Linux cung cấp hai phương pháp phi chuẩn để lấy danh sách tất cả IPC object:
- Các file trong thư mục `/proc/sysvipc`:
  - `/proc/sysvipc/msg`: Liệt kê tất cả message queue.
  - `/proc/sysvipc/sem`: Liệt kê tất cả semaphore set.
  - `/proc/sysvipc/shm`: Liệt kê tất cả shared memory segment.

---

## 45.8 Giới Hạn IPC

Kernel đặt giới hạn trên mỗi loại IPC object. Trên Linux, lệnh `ipcs -l` liệt kê các giới hạn. Các giới hạn có thể xem và thay đổi qua các file trong `/proc/sys/kernel/`.

---

## 45.9 Tóm Tắt

System V IPC gồm ba cơ chế: message queue, semaphore, và shared memory. Ba cơ chế này có nhiều điểm chung trong API: lời gọi `get` để tạo/mở object; integer key để nhận identifier; lời gọi `ctl` để xóa và thay đổi thuộc tính.

Thuật toán tạo identifier đảm bảo identifier mới sẽ khác với identifier cũ kể cả khi dùng cùng key, giúp ứng dụng client-server phát hiện server restart.

Lệnh `ipcs` liệt kê, `ipcrm` xóa System V IPC object. Các file trong `/proc/sysvipc` cung cấp thông tin về tất cả IPC object.

---

## 45.10 Bài Tập

- **45-1.** Viết chương trình xác minh rằng thuật toán `ftok()` dùng i-node number, minor device number, và `proj` value.
- **45-2.** Triển khai `ftok()`.
- **45-3.** Xác minh bằng thực nghiệm các nhận định về thuật toán tạo System V IPC identifier.
