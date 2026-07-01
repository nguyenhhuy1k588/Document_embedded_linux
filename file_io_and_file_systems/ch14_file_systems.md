## Chương 14
# Hệ Thống File (File Systems)

Trong các chương 4, 5, và 13, chúng ta đã xem xét file I/O, tập trung chủ yếu vào các file thông thường (tức là file trên đĩa). Trong chương này và các chương tiếp theo, chúng ta đi sâu vào nhiều chủ đề liên quan đến file:

- Chương này xem xét file system.
- Chương 15 mô tả các thuộc tính khác nhau liên quan đến file, bao gồm timestamp, quyền sở hữu, và quyền truy cập.
- Chương 16 và 17 xem xét hai tính năng mới của Linux 2.6: extended attribute và access control list (ACL).
- Chương 18 xem xét directory và link.

Phần lớn chương này liên quan đến file system — các tập hợp file và directory được tổ chức. Chúng ta giải thích nhiều khái niệm về file system, đôi khi dùng file system ext2 truyền thống của Linux làm ví dụ cụ thể. Chúng ta cũng mô tả ngắn gọn một số journaling file system có sẵn trên Linux.

Kết thúc chương là thảo luận về các system call dùng để mount và unmount file system, và các hàm thư viện dùng để lấy thông tin về các file system đã mount.

---

## 14.1 Device Special File (Thiết Bị)

Chương này thường đề cập đến disk device, vì vậy chúng ta bắt đầu với tổng quan ngắn gọn về khái niệm device file.

Một device special file tương ứng với một thiết bị trên hệ thống. Trong kernel, mỗi loại thiết bị có một device driver tương ứng, xử lý tất cả các yêu cầu I/O cho thiết bị. Device driver là một đơn vị kernel code triển khai một tập các thao tác thường tương ứng với các hành động input và output trên phần cứng liên quan. API do device driver cung cấp là cố định, và bao gồm các thao tác tương ứng với các system call `open()`, `close()`, `read()`, `write()`, `mmap()`, và `ioctl()`. Việc mỗi device driver cung cấp giao diện nhất quán, ẩn đi sự khác biệt trong hoạt động của từng thiết bị, cho phép tính universality của I/O (Mục 4.2).

Một số thiết bị là thực, như chuột, đĩa cứng, và ổ băng từ. Các thiết bị khác là ảo, nghĩa là không có phần cứng tương ứng; thay vào đó, kernel cung cấp (qua device driver) một thiết bị trừu tượng với API giống như thiết bị thực.

Thiết bị có thể được chia thành hai loại:

- **Character device** xử lý dữ liệu từng ký tự một. Terminal và bàn phím là các ví dụ về character device.
- **Block device** xử lý dữ liệu theo từng block. Kích thước block phụ thuộc vào loại thiết bị, nhưng thường là bội số của 512 byte. Đĩa cứng và ổ băng từ là ví dụ về block device.

Các device file xuất hiện trong file system, giống như các file khác, thường nằm dưới thư mục `/dev`. Superuser có thể tạo device file bằng lệnh `mknod`, và cùng tác vụ đó có thể được thực hiện trong một chương trình có đặc quyền (`CAP_MKNOD`) bằng system call `mknod()`.

> Chúng ta không mô tả chi tiết system call `mknod()` vì cách dùng của nó đơn giản, và mục đích duy nhất cần đến ngày nay là tạo device file — không phải là yêu cầu ứng dụng phổ biến. Chúng ta cũng có thể dùng `mknod()` để tạo FIFO (Mục 44.7), nhưng hàm `mkfifo()` được ưa thích hơn cho tác vụ này.

Trong các phiên bản Linux cũ hơn, `/dev` chứa các mục cho tất cả các thiết bị có thể có trên hệ thống, ngay cả khi thiết bị đó không thực sự được kết nối. Điều này có nghĩa `/dev` có thể chứa hàng nghìn mục không dùng đến, làm chậm quá trình quét directory của các chương trình, và khiến không thể dùng nội dung directory như một cách để khám phá các thiết bị thực sự có mặt. Trong Linux 2.6, các vấn đề này được giải quyết bởi chương trình `udev`. Chương trình `udev` dựa vào file system `sysfs`, xuất thông tin về thiết bị và các đối tượng kernel khác vào user space qua pseudo-file system được mount dưới `/sys`.

### Device ID

Mỗi device file có một major ID number và một minor ID number. Major ID xác định lớp chung của thiết bị, và được kernel dùng để tìm driver phù hợp cho loại thiết bị này. Minor ID xác định duy nhất một thiết bị cụ thể trong lớp chung. Major và minor ID của device file được hiển thị bởi lệnh `ls -l`.

Major và minor ID của thiết bị được ghi trong i-node cho device file. (Chúng ta mô tả i-node trong Mục 14.4.) Mỗi device driver đăng ký liên kết của mình với một major device ID cụ thể, và liên kết này cung cấp kết nối giữa device special file và device driver. Tên của device file không có liên quan khi kernel tìm kiếm device driver.

Trên Linux 2.4 và trước đó, tổng số thiết bị trên hệ thống bị hạn chế bởi thực tế là major và minor device ID mỗi cái chỉ được biểu diễn bằng 8 bit. Linux 2.6 giảm bớt hạn chế này bằng cách dùng nhiều bit hơn để giữ major và minor device ID (lần lượt là 12 và 20 bit).

---

## 14.2 Đĩa Cứng và Partition

Các file thông thường và directory thường nằm trên hard disk device. Trong các mục sau, chúng ta xem xét cách đĩa cứng được tổ chức và chia thành partition.

### Ổ đĩa cứng

Ổ đĩa cứng là thiết bị cơ học gồm một hoặc nhiều platter quay ở tốc độ cao (thường là hàng nghìn vòng/phút). Thông tin được mã hóa từ tính trên bề mặt đĩa được đọc hoặc sửa đổi bởi các đầu đọc/ghi di chuyển theo chiều hướng tâm. Về mặt vật lý, thông tin trên bề mặt đĩa nằm trên tập các vòng tròn đồng tâm gọi là track. Các track được chia thành một số sector, mỗi sector bao gồm một loạt physical block. Physical block thường có kích thước 512 byte (hoặc bội số của nó), và đại diện cho đơn vị thông tin nhỏ nhất mà ổ đĩa có thể đọc hoặc ghi.

Mặc dù các ổ đĩa hiện đại nhanh, việc đọc và ghi thông tin trên đĩa vẫn tốn nhiều thời gian. Đầu đĩa phải di chuyển đến track thích hợp (seek time), sau đó ổ đĩa phải chờ cho đến khi sector phù hợp quay dưới đầu đọc (rotational latency), và cuối cùng các block cần thiết phải được truyền (transfer time). Tổng thời gian cần để thực hiện thao tác như vậy thường là vài millisecond.

### Disk partition

Mỗi đĩa được chia thành một hoặc nhiều partition (không chồng lên nhau). Mỗi partition được kernel coi là một thiết bị riêng biệt nằm dưới thư mục `/dev`.

> Người quản trị hệ thống xác định số lượng, loại và kích thước của các partition trên đĩa bằng lệnh `fdisk`. Lệnh `fdisk -l` liệt kê tất cả các partition trên đĩa. File Linux-specific `/proc/partitions` liệt kê major và minor device number, kích thước, và tên của mỗi disk partition trên hệ thống.

Một disk partition có thể chứa bất kỳ loại thông tin nào, nhưng thường chứa một trong những thứ sau:

- Một file system chứa các file thông thường và directory, như mô tả trong Mục 14.3;
- Một data area được truy cập dưới dạng raw-mode device (một số hệ thống quản lý cơ sở dữ liệu dùng kỹ thuật này); hoặc
- Một swap area được kernel dùng cho quản lý bộ nhớ.

Swap area được tạo bằng lệnh `mkswap(8)`. Một process có đặc quyền (`CAP_SYS_ADMIN`) có thể dùng system call `swapon()` để thông báo cho kernel biết rằng một disk partition sẽ được dùng làm swap area. System call `swapoff()` thực hiện chức năng ngược lại.

> File Linux-specific `/proc/swaps` có thể được dùng để hiển thị thông tin về các swap area hiện đang được kích hoạt trên hệ thống.

---

## 14.3 File System

File system là tập hợp file thông thường và directory được tổ chức. File system được tạo bằng lệnh `mkfs`.

Một trong những điểm mạnh của Linux là nó hỗ trợ nhiều loại file system, bao gồm:

- File system ext2 truyền thống;
- Nhiều file system UNIX gốc như Minix, System V, và BSD;
- Các file system của Microsoft: FAT, FAT32, và NTFS;
- File system ISO 9660 CD-ROM;
- HFS của Apple Macintosh;
- Nhiều network file system, gồm NFS của Sun, SMB của IBM và Microsoft, NCP của Novell, và Coda; và
- Nhiều journaling file system, gồm ext3, ext4, Reiserfs, JFS, XFS, và Btrfs.

Các loại file system hiện được kernel nhận biết có thể xem trong file Linux-specific `/proc/filesystems`.

> Linux 2.6.14 đã thêm cơ sở Filesystem in Userspace (FUSE). Cơ chế này thêm các hook vào kernel cho phép file system được triển khai hoàn toàn thông qua chương trình user-space, không cần patch hay recompile kernel.

### File system ext2

Trong nhiều năm, file system được dùng rộng rãi nhất trên Linux là ext2 (Second Extended File System), kế thừa file system Linux gốc ext. Trong thời gian gần đây, việc dùng ext2 đã giảm dần do sự ưa chuộng các journaling file system.

### Cấu trúc file system

Đơn vị cơ bản để cấp phát không gian trong file system là logical block, là bội số của các physical block liên tiếp trên disk device. Ví dụ, kích thước logical block trên ext2 là 1024, 2048, hoặc 4096 byte.

```
┌──────────┬──────────┬──────────┐
Đĩa  │partition │partition │partition │
     └────┬─────┴─────┬────┴─────┬────┘
          │           │          │
          └─ ─ ─ ─ ─ ─┼─ ─ ─ ─ ─ ┘
                      │
     ┌────────┬───────┴──┬────────┬────────────────────────┐
File │  boot  │  super-  │ i-node │     data blocks        │
sys  │  block │  block   │ table  │                        │
     └────────┴──────────┴────────┴────────────────────────┘
```

**Hình 14-1:** Bố cục disk partition và file system

Một file system chứa các phần sau:

- **Boot block**: Đây luôn là block đầu tiên trong file system. Boot block không được file system dùng; thay vào đó, nó chứa thông tin dùng để khởi động hệ điều hành. Mặc dù chỉ cần một boot block, tất cả file system đều có boot block (hầu hết không được dùng đến).

- **Superblock**: Đây là một block duy nhất, ngay sau boot block, chứa thông tin về tham số của file system, bao gồm:
  - Kích thước của i-node table;
  - Kích thước logical block trong file system này; và
  - Kích thước file system tính bằng logical block.

- **I-node table**: Mỗi file hoặc directory trong file system có một mục duy nhất trong i-node table. Mục này ghi lại nhiều thông tin về file. I-node table đôi khi còn gọi là i-list.

- **Data block**: Phần lớn không gian trong file system được dùng cho các data block tạo nên các file và directory trong file system.

---

## 14.4 I-node

I-node table của file system chứa một i-node (viết tắt của index node) cho mỗi file nằm trong file system. I-node được nhận dạng bằng số theo vị trí tuần tự trong i-node table. I-node number (hay i-number) của file là trường đầu tiên được hiển thị bởi lệnh `ls -li`. Thông tin được lưu trong i-node bao gồm:

- Loại file (ví dụ: file thông thường, directory, symbolic link, character device).
- Owner (còn gọi là user ID hay UID) của file.
- Group (còn gọi là group ID hay GID) của file.
- Quyền truy cập cho ba loại người dùng: owner (đôi khi gọi là user), group, và other.
- Ba timestamp: thời gian truy cập file lần cuối (`ls -lu`), thời gian sửa đổi file lần cuối (mặc định hiển thị bởi `ls -l`), và thời gian thay đổi trạng thái lần cuối (thay đổi thông tin i-node, hiển thị bởi `ls -lc`). Đáng chú ý là hầu hết các file system Linux không ghi lại thời gian tạo file.
- Số hard link đến file.
- Kích thước file tính bằng byte.
- Số block thực sự được cấp phát cho file, tính bằng đơn vị block 512 byte.
- Pointer đến các data block của file.

### I-node và data block pointer trong ext2

Giống như hầu hết các file system UNIX, ext2 không lưu các data block của file theo cách liên tiếp hay theo thứ tự tuần tự (mặc dù cố gắng lưu gần nhau). Để định vị các data block của file, kernel duy trì một tập pointer trong i-node.

Trong ext2, mỗi i-node chứa 15 pointer. 12 pointer đầu tiên (số 0 đến 11) trỏ đến vị trí trong file system của 12 block đầu tiên của file. Pointer tiếp theo là pointer đến một block pointer trỏ đến vị trí của block thứ 13 trở đi. Đây được gọi là **indirect pointer block** (IPB). Với block size 4096 byte, có thể có tới 1024 pointer trong block này.

Pointer thứ 14 (số 13) là **double indirect pointer** (2IPB) — trỏ đến các block pointer, các pointer này lại trỏ đến các block pointer khác, cuối cùng trỏ đến data block của file. Cuối cùng, pointer cuối cùng trong i-node là **triple indirect pointer** (3IPB).

Hệ thống phức tạp này được thiết kế để đáp ứng nhiều yêu cầu: cho phép structure i-node có kích thước cố định trong khi vẫn hỗ trợ file có kích thước tùy ý; cho phép file system lưu các block không liên tiếp, đồng thời cho phép truy cập ngẫu nhiên qua `lseek()`; và với các file nhỏ (chiếm đa số), cho phép truy cập nhanh qua các direct pointer của i-node.

Thiết kế này cũng cho phép kích thước file rất lớn; với block size 4096 byte, kích thước file lý thuyết lớn nhất là hơn 4 terabyte. Ngoài ra, các file có thể có hole (như mô tả trong Mục 4.7) — thay vì cấp phát block toàn null byte cho hole, file system có thể đánh dấu các pointer tương ứng bằng 0 để chỉ ra rằng chúng không tham chiếu đến disk block thực sự.

---

## 14.5 Virtual File System (VFS)

Mỗi file system có sẵn trên Linux khác nhau ở chi tiết triển khai. Chẳng hạn, cách cấp phát block hay cách tổ chức directory là khác nhau. Nếu mọi chương trình làm việc với file cần hiểu chi tiết cụ thể của từng file system, việc viết chương trình làm việc với tất cả các file system khác nhau sẽ gần như không thể. Virtual file system (VFS, đôi khi còn gọi là virtual file switch) là tính năng kernel giải quyết vấn đề này bằng cách tạo lớp trừu tượng cho các thao tác file system:

- VFS định nghĩa giao diện chung cho các thao tác file system. Tất cả chương trình làm việc với file chỉ định các thao tác bằng giao diện chung này.
- Mỗi file system cung cấp một triển khai cho giao diện VFS.

Theo cơ chế này, các chương trình chỉ cần hiểu giao diện VFS và có thể bỏ qua chi tiết của từng triển khai file system cụ thể.

Giao diện VFS bao gồm các thao tác tương ứng với tất cả system call thông thường để làm việc với file system và directory, như `open()`, `read()`, `write()`, `lseek()`, `close()`, `truncate()`, `stat()`, `mount()`, `umount()`, `mmap()`, `mkdir()`, `link()`, `unlink()`, `symlink()`, và `rename()`.

```
Ứng dụng
    ↓
Virtual File System (VFS)
    ↓         ↓         ↓         ↓        ↓
  ext2      ext3    Reiserfs   VFAT      NFS
```

**Hình 14-3:** Virtual file system

---

## 14.6 Journaling File System

File system ext2 là ví dụ điển hình về file system UNIX truyền thống, và chịu một hạn chế cổ điển: sau khi hệ thống bị crash, phải thực hiện kiểm tra tính nhất quán của file system (`fsck`) khi khởi động lại để đảm bảo tính toàn vẹn. Điều này cần thiết vì tại thời điểm crash, việc cập nhật file có thể chỉ mới hoàn thành một phần, và metadata của file system (directory entry, thông tin i-node, và file data block pointer) có thể ở trạng thái không nhất quán.

Vấn đề là kiểm tra tính nhất quán đòi hỏi kiểm tra toàn bộ file system. Trên file system lớn, việc này có thể mất nhiều giờ — rất nghiêm trọng với các hệ thống cần tính sẵn sàng cao.

Journaling file system loại bỏ sự cần thiết phải kiểm tra tính nhất quán kéo dài sau khi crash. Journaling file system ghi nhật ký (journal) tất cả cập nhật metadata vào một file nhật ký đặc biệt trước khi thực sự thực hiện chúng. Các cập nhật được ghi theo các nhóm cập nhật metadata liên quan (transaction). Trong trường hợp crash giữa chừng một transaction, khi khởi động lại, log có thể được dùng để nhanh chóng redo bất kỳ cập nhật nào chưa hoàn thành và đưa file system về trạng thái nhất quán. Ngay cả journaling file system rất lớn thường có thể sẵn sàng trong vài giây sau crash.

Nhược điểm đáng chú ý nhất của journaling là nó thêm thời gian vào các cập nhật file, mặc dù thiết kế tốt có thể làm cho overhead này thấp.

Các journaling file system có sẵn cho Linux bao gồm:

- **Reiserfs**: File system journaling đầu tiên được tích hợp vào kernel (phiên bản 2.4.1). Reiserfs cung cấp tính năng tail packing: các file nhỏ (và phần cuối của các file lớn hơn) được đóng gói vào cùng disk block với metadata file, tiết kiệm đáng kể không gian đĩa.
- **ext3**: Kết quả của dự án thêm journaling vào ext2 với tác động tối thiểu. Đường di chuyển từ ext2 sang ext3 rất dễ (không cần backup và restore). Được tích hợp vào kernel phiên bản 2.4.15.
- **JFS**: Được phát triển bởi IBM. Tích hợp vào kernel 2.4.20.
- **XFS**: Được phát triển bởi Silicon Graphics (SGI) đầu thập niên 1990 cho Irix. Port sang Linux năm 2001. Tích hợp vào kernel 2.4.24.
- **ext4**: Kế thừa của ext3. Các tính năng được lên kế hoạch bao gồm extent (cấp phát các block liên tiếp), kiểm tra file system nhanh hơn, và hỗ trợ nanosecond timestamp.
- **Btrfs**: File system mới được thiết kế từ đầu với nhiều tính năng hiện đại, gồm extent, writable snapshot, checksum trên data và metadata, và kiểm tra file system online. Tích hợp vào kernel 2.6.29.

---

## 14.7 Cây Thư Mục Đơn và Mount Point

Trên Linux, cũng như các hệ thống UNIX khác, tất cả file từ tất cả file system đều nằm dưới một cây thư mục duy nhất. Gốc của cây này là thư mục root, `/` (slash). Các file system khác được mount dưới thư mục root và xuất hiện như các subtree trong cây phân cấp tổng thể. Superuser dùng lệnh sau để mount file system:

```
$ mount device directory
```

Lệnh này gắn file system trên thiết bị được đặt tên vào cây phân cấp thư mục tại directory được chỉ định — mount point của file system. Có thể thay đổi vị trí mount bằng cách unmount rồi mount lại tại điểm khác.

> Từ Linux 2.4.19 trở đi, kernel hỗ trợ per-process mount namespace. Điều này có nghĩa mỗi process có thể có tập mount point riêng, và do đó có thể thấy cây thư mục đơn khác với các process khác.

Để liệt kê các file system hiện đang mount, dùng lệnh `mount` không có đối số:

```
$ mount
/dev/sda6 on / type ext4 (rw)
proc on /proc type proc (rw)
sysfs on /sys type sysfs (rw)
/dev/sda8 on /home type ext3 (rw,acl,user_xattr)
/dev/sda1 on /windows/C type vfat (rw,noexec,nosuid,nodev)
/dev/sda9 on /home/mtk/test type reiserfs (rw)
```

---

## 14.8 Mount và Unmount File System

Các system call `mount()` và `umount()` cho phép process có đặc quyền (`CAP_SYS_ADMIN`) mount và unmount file system.

Trước khi xem xét các system call này, cần biết về ba file chứa thông tin về các file system đã mount hoặc có thể mount:

- Danh sách các file system đang mount có thể đọc từ file virtual Linux-specific `/proc/mounts`. `/proc/mounts` là giao diện đến các cấu trúc dữ liệu kernel, vì vậy nó luôn chứa thông tin chính xác về các file system đã mount.
- Các lệnh `mount(8)` và `umount(8)` tự động duy trì file `/etc/mtab`, chứa thông tin tương tự như trong `/proc/mounts`, nhưng chi tiết hơn một chút.
- File `/etc/fstab`, được người quản trị hệ thống duy trì thủ công, chứa mô tả về tất cả các file system có sẵn trên hệ thống.

Các file `/proc/mounts`, `/etc/mtab`, và `/etc/fstab` chia sẻ định dạng chung, được mô tả trong manual page `fstab(5)`. Ví dụ một dòng từ file `/proc/mounts`:

```
/dev/sda9 /boot ext3 rw 0 0
```

Dòng này chứa sáu trường:
1. Tên thiết bị được mount.
2. Mount point cho thiết bị.
3. Loại file system.
4. Mount flag. Trong ví dụ trên, `rw` chỉ ra file system được mount read-write.
5. Số được dùng để kiểm soát thao tác backup của file system bởi `dump(8)`.
6. Số được dùng để kiểm soát thứ tự kiểm tra file system bởi `fsck(8)` lúc khởi động.

### 14.8.1 Mount File System: mount()

System call `mount()` mount file system chứa trên thiết bị được chỉ định bởi `source` dưới directory (mount point) được chỉ định bởi `target`.

```c
#include <sys/mount.h>
int mount(const char *source, const char *target, const char *fstype,
          unsigned long mountflags, const void *data);
                                           Returns 0 on success, or –1 on error
```

Đối số `fstype` là chuỗi xác định loại file system, như `ext4` hay `btrfs`.

Đối số `mountflags` là bit mask được xây dựng bằng cách OR (`|`) không hoặc nhiều hơn các flag trong Bảng 14-1.

Đối số cuối `data` là pointer đến buffer thông tin có cách diễn giải phụ thuộc vào file system. Đối với hầu hết các loại file system, đây là chuỗi bao gồm các cài đặt tùy chọn phân cách bằng dấu phẩy.

**Bảng 14-1:** Giá trị mountflags cho `mount()`

| Flag | Mục đích |
|------|---------|
| `MS_BIND` | Tạo bind mount (từ Linux 2.4) |
| `MS_DIRSYNC` | Làm cho cập nhật directory đồng bộ (từ Linux 2.6) |
| `MS_MANDLOCK` | Cho phép mandatory locking trên file |
| `MS_MOVE` | Di chuyển mount point sang vị trí mới theo cách atomic |
| `MS_NOATIME` | Không cập nhật last access time cho file |
| `MS_NODEV` | Không cho phép truy cập thiết bị |
| `MS_NODIRATIME` | Không cập nhật last access time cho directory |
| `MS_NOEXEC` | Không cho phép thực thi chương trình |
| `MS_NOSUID` | Vô hiệu hóa chương trình set-user-ID và set-group-ID |
| `MS_RDONLY` | Mount read-only; không thể tạo hay sửa đổi file |
| `MS_REC` | Mount đệ quy (từ Linux 2.6.20) |
| `MS_RELATIME` | Chỉ cập nhật last access time nếu cũ hơn last modification hoặc last status change time |
| `MS_REMOUNT` | Remount với mountflags và data mới |
| `MS_STRICTATIME` | Luôn cập nhật last access time (từ Linux 2.6.30) |
| `MS_SYNCHRONOUS` | Làm cho tất cả cập nhật file và directory đồng bộ |

Mô tả một số flag quan trọng:

**MS_BIND**: Tạo bind mount. Xem Mục 14.9.4.

**MS_DIRSYNC**: Làm cho cập nhật directory đồng bộ. Tương tự hiệu ứng của flag `O_SYNC` của `open()`, nhưng chỉ áp dụng cho cập nhật directory.

**MS_NOATIME**: Không cập nhật last access time cho file trong file system này. Mục đích là loại bỏ disk access bổ sung cần để cập nhật file i-node mỗi khi file được truy cập.

**MS_NODEV**: Không cho phép truy cập block và character device trên file system này. Đây là tính năng bảo mật.

**MS_NOEXEC**: Không cho phép thực thi chương trình hay script từ file system này.

**MS_NOSUID**: Vô hiệu hóa set-user-ID và set-group-ID trên file system này. Tính năng bảo mật ngăn người dùng chạy các chương trình set-user-ID/set-group-ID từ thiết bị di động.

**MS_RDONLY**: Mount file system read-only.

**MS_REMOUNT**: Thay đổi mountflags và data cho file system đã được mount (ví dụ: để làm cho file system read-only trở nên writable).

**MS_SYNCHRONOUS**: Làm cho tất cả cập nhật file và directory trên file system này đồng bộ.

**Chương trình ví dụ**

Listing 14-1 cung cấp giao diện command-line cho system call `mount(2)`. Về cơ bản, nó là phiên bản thô sơ của lệnh `mount(8)`.

```
$ su
Password:
# mkdir /testfs
# ./t_mount -t ext2 -o bsdgroups /dev/sda12 /testfs
# cat /proc/mounts | grep sda12
/dev/sda12 /testfs ext3 rw 0 0
# ./t_mount -f Rr /dev/sda12 /testfs     Remount read-only
# cat /proc/mounts | grep sda12
/dev/sda12 /testfs ext3 ro 0 0
# mkdir /demo
# ./t_mount -f m /testfs /demo           Di chuyển mount point
# cat /proc/mounts | grep sda12
/dev/sda12 /demo ext3 ro 0
```

**Listing 14-1:** Sử dụng `mount()`

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– filesys/t_mount.c
#include <sys/mount.h>
#include "tlpi_hdr.h"
static void
usageError(const char *progName, const char *msg)
{
    if (msg != NULL)
        fprintf(stderr, "%s", msg);
    fprintf(stderr, "Usage: %s [options] source target\n\n", progName);
    fprintf(stderr, "Available options:\n");
#define fpe(str) fprintf(stderr, "  " str)
    fpe("-t fstype [e.g., 'ext2' or 'reiserfs']\n");
    fpe("-o data [file system-dependent options]\n");
    fpe("-f mountflags can include any of:\n");
#define fpe2(str) fprintf(stderr, "    " str)
    fpe2("b - MS_BIND        create a bind mount\n");
    fpe2("d - MS_DIRSYNC     synchronous directory updates\n");
    fpe2("l - MS_MANDLOCK    permit mandatory locking\n");
    fpe2("m - MS_MOVE        atomically move subtree\n");
    fpe2("A - MS_NOATIME     don't update atime\n");
    fpe2("V - MS_NODEV       don't permit device access\n");
    fpe2("D - MS_NODIRATIME  don't update atime on directories\n");
    fpe2("E - MS_NOEXEC      don't allow executables\n");
    fpe2("S - MS_NOSUID      disable set-user/group-ID programs\n");
    fpe2("r - MS_RDONLY      read-only mount\n");
    fpe2("c - MS_REC         recursive mount\n");
    fpe2("R - MS_REMOUNT     remount\n");
    fpe2("s - MS_SYNCHRONOUS make writes synchronous\n");
    exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
    unsigned long flags;
    char *data, *fstype;
    int j, opt;
    flags = 0; data = NULL; fstype = NULL;
    while ((opt = getopt(argc, argv, "o:t:f:")) != -1) {
        switch (opt) {
        case 'o': data = optarg;   break;
        case 't': fstype = optarg; break;
        case 'f':
            for (j = 0; j < strlen(optarg); j++) {
                switch (optarg[j]) {
                case 'b': flags |= MS_BIND;        break;
                case 'd': flags |= MS_DIRSYNC;     break;
                case 'l': flags |= MS_MANDLOCK;    break;
                case 'm': flags |= MS_MOVE;        break;
                case 'A': flags |= MS_NOATIME;     break;
                case 'V': flags |= MS_NODEV;       break;
                case 'D': flags |= MS_NODIRATIME;  break;
                case 'E': flags |= MS_NOEXEC;      break;
                case 'S': flags |= MS_NOSUID;      break;
                case 'r': flags |= MS_RDONLY;      break;
                case 'c': flags |= MS_REC;         break;
                case 'R': flags |= MS_REMOUNT;     break;
                case 's': flags |= MS_SYNCHRONOUS; break;
                default:  usageError(argv[0], NULL);
                }
            }
            break;
        default: usageError(argv[0], NULL);
        }
    }
    if (argc != optind + 2)
        usageError(argv[0], "Wrong number of arguments\n");
    if (mount(argv[optind], argv[optind + 1], fstype, flags, data) == -1)
        errExit("mount");
    exit(EXIT_SUCCESS);
}
––––––––––––––––––––––––––––––––––––––––––––––––––––––––– filesys/t_mount.c
```

### 14.8.2 Unmount File System: umount() và umount2()

System call `umount()` unmount một file system đã mount.

```c
#include <sys/mount.h>
int umount(const char *target);
                                             Returns 0 on success, or –1 on error
```

Đối số `target` chỉ định mount point của file system cần unmount.

Không thể unmount file system đang bận (busy): tức là có file đang mở trên file system, hoặc current working directory của một process nằm trong file system. Gọi `umount()` trên file system bận sẽ trả về lỗi `EBUSY`.

System call `umount2()` là phiên bản mở rộng của `umount()`, cho phép kiểm soát tinh hơn thông qua đối số `flags`.

```c
#include <sys/mount.h>
int umount2(const char *target, int flags);
                                             Returns 0 on success, or –1 on error
```

Đối số `flags` là bit mask gồm không hoặc nhiều giá trị sau OR với nhau:

- **`MNT_DETACH` (từ Linux 2.4.11)**: Thực hiện lazy unmount. Mount point được đánh dấu để không process nào có thể thực hiện truy cập mới, nhưng các process đang dùng nó vẫn có thể tiếp tục. File system thực sự unmount khi tất cả process ngừng dùng mount.
- **`MNT_EXPIRE` (từ Linux 2.6.8)**: Đánh dấu mount point là expired. Cung cấp cơ chế để unmount file system không được dùng trong một khoảng thời gian.
- **`MNT_FORCE`**: Buộc unmount ngay cả khi thiết bị bận (chỉ cho NFS mount). Có thể gây mất dữ liệu.
- **`UMOUNT_NOFOLLOW` (từ Linux 2.6.34)**: Không dereference `target` nếu nó là symbolic link.

---

## 14.9 Các Tính Năng Mount Nâng Cao

### 14.9.1 Mount File System tại Nhiều Mount Point

Từ kernel 2.4, file system có thể được mount tại nhiều vị trí trong file system hierarchy. Vì mỗi mount point hiển thị cùng một subtree, các thay đổi thực hiện qua một mount point có thể nhìn thấy qua các mount point khác:

```
# mount /dev/sda12 /testfs
# mount /dev/sda12 /demo
# touch /testfs/myfile
# ls /demo
lost+found  myfile
```

### 14.9.2 Stack Nhiều Mount trên Cùng Mount Point

Từ kernel 2.4, Linux cho phép stack nhiều mount trên một mount point duy nhất. Mỗi mount mới ẩn đi subtree directory đã nhìn thấy trước đó tại mount point đó. Khi mount trên cùng stack bị unmount, mount bị ẩn trước đó lại trở nên nhìn thấy:

```
# mount /dev/sda12 /testfs
# mount /dev/sda13 /testfs    Stack mount thứ hai
# touch /testfs/newfile
# umount /testfs              Bỏ mount khỏi stack
# ls /testfs                  Mount trước đó giờ nhìn thấy được
lost+found  myfile
```

### 14.9.3 Mount Flag là Tùy Chọn Per-Mount

Từ kernel 2.4, vì không còn tương quan 1-1 giữa file system và mount point, một số flag có thể được đặt theo từng mount riêng lẻ: `MS_NOATIME`, `MS_NODEV`, `MS_NODIRATIME`, `MS_NOEXEC`, `MS_NOSUID`, `MS_RDONLY`, và `MS_RELATIME`.

### 14.9.4 Bind Mount

Từ kernel 2.4, Linux cho phép tạo bind mount. Bind mount (tạo bằng flag `MS_BIND` của `mount()`) cho phép directory hoặc file được mount tại một vị trí khác trong file-system hierarchy. Điều này dẫn đến directory hay file có thể nhìn thấy từ cả hai vị trí. Bind mount có một số điểm khác với hard link:

- Bind mount có thể vượt qua các mount point của file system (và ngay cả chroot jail).
- Có thể tạo bind mount cho directory.

```
# mkdir d1
# touch d1/x
# mkdir d2
# mount --bind d1 d2    Tạo bind mount: d1 nhìn thấy qua d2
# ls d2
x
# touch d2/y
# ls d1
x  y
```

Một ứng dụng của bind mount là khi tạo chroot jail (Mục 18.12): thay vì sao chép các directory chuẩn (như `/lib`) vào jail, ta chỉ cần tạo bind mount cho các directory đó (có thể mount read-only) trong jail.

### 14.9.5 Recursive Bind Mount

Theo mặc định, nếu tạo bind mount cho directory bằng `MS_BIND`, chỉ directory đó được mount tại vị trí mới; các submount bên dưới directory nguồn không được nhân bản. Linux 2.4.11 thêm flag `MS_REC` có thể được OR với `MS_BIND` để submount được nhân bản dưới mount target (recursive bind mount). Lệnh `mount(8)` cung cấp tùy chọn `--rbind` cho cùng hiệu ứng.

---

## 14.10 Virtual Memory File System: tmpfs

Ngoài các file system trên đĩa, Linux còn hỗ trợ các virtual file system nằm trong memory. Đối với ứng dụng, chúng trông giống như bất kỳ file system nào khác, nhưng có một điểm khác biệt quan trọng: các thao tác file nhanh hơn nhiều vì không có truy cập đĩa.

File system memory-based tinh vi nhất hiện nay là **tmpfs**, xuất hiện lần đầu trong Linux 2.4. tmpfs khác với các file system memory-based khác ở chỗ nó là virtual memory file system — dùng không chỉ RAM mà còn swap space nếu RAM cạn kiệt.

Để tạo tmpfs file system:

```
# mount -t tmpfs source target
```

`source` có thể là bất kỳ tên nào; ý nghĩa của nó chỉ là nó xuất hiện trong `/proc/mounts`. Không cần dùng `mkfs` để tạo file system trước vì kernel tự động xây dựng file system như một phần của lời gọi `mount()`.

```
# mount -t tmpfs newtmp /tmp
# cat /proc/mounts | grep tmp
newtmp /tmp tmpfs rw 0 0
```

Theo mặc định, tmpfs file system được phép lớn đến nửa kích thước RAM, nhưng tùy chọn `size=nbytes` có thể được dùng để đặt mức trần khác. Nếu tmpfs file system bị unmount, hoặc hệ thống bị crash, tất cả dữ liệu trong file system sẽ bị mất.

Ngoài việc dùng bởi ứng dụng người dùng, tmpfs file system cũng phục vụ hai mục đích đặc biệt:

- Một tmpfs file system vô hình, được kernel mount nội bộ, được dùng để triển khai System V shared memory (Chương 48) và shared anonymous memory mapping (Chương 49).
- Một tmpfs file system được mount tại `/dev/shm` được dùng cho việc triển khai POSIX shared memory và POSIX semaphore trong glibc.

---

## 14.11 Lấy Thông Tin về File System: statvfs()

Các hàm thư viện `statvfs()` và `fstatvfs()` lấy thông tin về một file system đã mount.

```c
#include <sys/statvfs.h>
int statvfs(const char *pathname, struct statvfs *statvfsbuf);
int fstatvfs(int fd, struct statvfs *statvfsbuf);
                                         Both return 0 on success, or –1 on error
```

Điểm khác biệt duy nhất giữa hai hàm này là cách xác định file system. Với `statvfs()`, dùng `pathname` để chỉ định tên bất kỳ file nào trong file system. Với `fstatvfs()`, chỉ định qua file descriptor đã mở `fd`. Cả hai hàm trả về structure `statvfs` chứa thông tin về file system:

```c
struct statvfs {
    unsigned long f_bsize;   /* File-system block size (in bytes) */
    unsigned long f_frsize;  /* Fundamental file-system block size (in bytes) */
    fsblkcnt_t    f_blocks;  /* Total number of blocks (in units of 'f_frsize') */
    fsblkcnt_t    f_bfree;   /* Total number of free blocks */
    fsblkcnt_t    f_bavail;  /* Free blocks available to unprivileged process */
    fsfilcnt_t    f_files;   /* Total number of i-nodes */
    fsfilcnt_t    f_ffree;   /* Total number of free i-nodes */
    fsfilcnt_t    f_favail;  /* I-nodes available to unprivileged process */
    unsigned long f_fsid;    /* File-system ID */
    unsigned long f_flag;    /* Mount flags */
    unsigned long f_namemax; /* Maximum length of filenames on this file system */
};
```

Một số điểm đáng lưu ý:

- Nhiều file system Linux gốc hỗ trợ reserved block — một phần block được dành cho superuser, để khi file system đầy, superuser vẫn có thể đăng nhập và giải quyết vấn đề. Sự khác biệt giữa `f_bfree` và `f_bavail` cho biết có bao nhiêu block được dành.
- Trường `f_flag` là bit mask của các flag dùng để mount file system, tương tự `mountflags` của `mount(2)`, nhưng dùng hằng số với tên bắt đầu bằng `ST_` thay vì `MS_`.

---

## 14.12 Tóm Tắt

Thiết bị được biểu diễn bởi các mục trong thư mục `/dev`. Mỗi thiết bị có device driver tương ứng, triển khai một tập thao tác chuẩn, gồm những thao tác tương ứng với các system call `open()`, `read()`, `write()`, và `close()`.

Đĩa cứng được chia thành một hoặc nhiều partition, mỗi partition có thể chứa một file system. File system là tập hợp file thông thường và directory được tổ chức. Linux triển khai nhiều loại file system, gồm file system ext2 truyền thống. File system ext2 bao gồm boot block, superblock, i-node table, và data area chứa các data block. Mỗi file có một mục trong i-node table chứa nhiều thông tin về file.

Linux cung cấp nhiều journaling file system, gồm Reiserfs, ext3, ext4, XFS, JFS, và Btrfs. Journaling file system ghi các cập nhật metadata vào log file trước khi thực hiện cập nhật thực sự, cho phép nhanh chóng khôi phục về trạng thái nhất quán sau crash.

Tất cả file system trên hệ thống Linux được mount dưới một cây thư mục duy nhất, với thư mục `/` làm root. Process có đặc quyền có thể mount và unmount file system bằng các system call `mount()` và `umount()`. Thông tin về file system đã mount có thể lấy bằng `statvfs()`.

---

## 14.13 Bài Tập

**14-1.** Viết chương trình đo thời gian cần thiết để tạo rồi xóa nhiều file 1 byte từ một directory duy nhất. Chương trình nên tạo file có tên dạng `xNNNNNN`, trong đó `NNNNNN` được thay bằng số ngẫu nhiên sáu chữ số. Các file nên được tạo theo thứ tự ngẫu nhiên mà tên được tạo ra, rồi xóa theo thứ tự số tăng dần. Số file và directory nên có thể chỉ định trên command line. Đo thời gian cần thiết cho các giá trị khác nhau và các file system khác nhau.
