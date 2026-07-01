## Chương 16
# Extended Attribute

Chương này mô tả extended attribute (EA) — tính năng cho phép gắn metadata tùy ý, dưới dạng các cặp tên-giá trị, vào i-node của file. EA được thêm vào Linux trong phiên bản 2.6.

---

## 16.1 Tổng Quan

EA được dùng để triển khai access control list (Chương 17) và file capability (Chương 39). Tuy nhiên, thiết kế của EA đủ tổng quát để cho phép dùng cho các mục đích khác, ví dụ: ghi lại phiên bản file, thông tin về MIME type hoặc character set, hoặc con trỏ đến icon đồ họa.

EA không được quy định trong SUSv3. EA yêu cầu hỗ trợ từ file system bên dưới. Hỗ trợ này được cung cấp trong Btrfs, ext2, ext3, ext4, JFS, Reiserfs, và XFS.

### EA namespace

EA có tên dạng `namespace.name`. Thành phần `namespace` phân tách EA thành các lớp có chức năng khác nhau. Thành phần `name` xác định duy nhất một EA trong namespace đã cho.

Bốn giá trị được hỗ trợ cho namespace: `user`, `trusted`, `system`, và `security`.

- **User EA**: Có thể được thao tác bởi process không có đặc quyền, tùy thuộc vào kiểm tra quyền file: để lấy giá trị của user EA cần quyền read trên file; để thay đổi cần quyền write. Để gắn user EA với file trên ext2, ext3, ext4, hoặc Reiserfs, file system bên dưới phải được mount với tùy chọn `user_xattr`:

  ```
  $ mount -o user_xattr device directory
  ```

- **Trusted EA**: Giống user EA nhưng process phải có đặc quyền (`CAP_SYS_ADMIN`) để thao tác.

- **System EA**: Được kernel dùng để gắn các đối tượng hệ thống với file. Hiện tại, loại đối tượng duy nhất được hỗ trợ là access control list (Chương 17).

- **Security EA**: Được dùng để lưu security label của file cho các module bảo mật của hệ điều hành, và để gắn capability với file thực thi (Mục 39.3.2).

Một i-node có thể có nhiều EA liên quan, trong cùng namespace hoặc các namespace khác nhau. Tên EA trong mỗi namespace là các tập riêng biệt. Trong namespace `user` và `trusted`, tên EA có thể là chuỗi tùy ý. Trong namespace `system`, chỉ cho phép các tên được kernel cho phép rõ ràng.

### Tạo và xem EA từ shell

Từ shell, ta có thể dùng lệnh `setfattr(1)` và `getfattr(1)` để đặt và xem EA trên file:

```
$ touch tfile
$ setfattr -n user.x -v "The past is not dead." tfile
$ setfattr -n user.y -v "In fact, it's not even past." tfile
$ getfattr -n user.x tfile          Lấy giá trị của một EA đơn
# file: tfile
user.x="The past is not dead."

$ getfattr -d tfile                  Dump giá trị của tất cả user EA
# file: tfile
user.x="The past is not dead."
user.y="In fact, it's not even past."
$ setfattr -n user.x tfile           Đổi giá trị EA thành chuỗi rỗng
$ getfattr -d tfile
# file: tfile
user.x
user.y="In fact, it's not even past."
$ setfattr -x user.y tfile           Xóa một EA
$ getfattr -d tfile
# file: tfile
user.x
```

Một điểm đáng chú ý: giá trị của EA có thể là chuỗi rỗng, khác với EA chưa được định nghĩa. Theo mặc định, `getfattr` chỉ liệt kê giá trị của user EA. Để liệt kê tất cả EA trên file:

```
$ getfattr -m - file
```

---

## 16.2 Chi Tiết Triển Khai Extended Attribute

### Hạn chế trên user extended attribute

Chỉ có thể đặt user EA trên file và directory, không phải loại file khác vì:

- Với symbolic link, tất cả quyền được bật cho tất cả người dùng và không thể thay đổi — không thể dùng quyền để ngăn user tùy ý đặt EA trên symbolic link.
- Với device file, socket, và FIFO, quyền kiểm soát truy cập I/O cho đối tượng bên dưới, thao tác quyền để kiểm soát tạo EA sẽ xung đột với mục đích này.

Hơn nữa, process không có đặc quyền không thể đặt user EA trên directory thuộc sở hữu của user khác nếu sticky bit được đặt trên directory đó.

### Giới hạn triển khai

Linux VFS áp đặt các giới hạn sau trên EA trên tất cả file system:

- Độ dài tên EA bị giới hạn ở 255 ký tự.
- Giá trị EA bị giới hạn ở 64 kB.

Ngoài ra, một số file system áp đặt giới hạn nghiêm ngặt hơn:
- Trên ext2, ext3, và ext4: tổng số byte dùng bởi tên và giá trị của tất cả EA trên một file bị giới hạn ở kích thước một logical disk block (1024, 2048, hoặc 4096 byte).
- Trên JFS: giới hạn trên 128 kB cho tổng byte dùng bởi EA trên một file.

---

## 16.3 System Call Để Thao Tác Extended Attribute

### Tạo và sửa đổi EA

Các system call `setxattr()`, `lsetxattr()`, và `fsetxattr()` đặt giá trị của một EA của file.

```c
#include <sys/xattr.h>
int setxattr(const char *pathname, const char *name, const void *value,
             size_t size, int flags);
int lsetxattr(const char *pathname, const char *name, const void *value,
              size_t size, int flags);
int fsetxattr(int fd, const char *name, const void *value,
              size_t size, int flags);
                                      All return 0 on success, or –1 on error
```

Sự khác biệt giữa ba lời gọi này tương tự như giữa `stat()`, `lstat()`, và `fstat()`:
- `setxattr()`: xác định file theo pathname, dereference symbolic link;
- `lsetxattr()`: xác định theo pathname, không dereference symbolic link; và
- `fsetxattr()`: xác định file qua file descriptor `fd`.

Đối số `name` là chuỗi kết thúc null xác định tên EA. Đối số `value` là pointer đến buffer định nghĩa giá trị mới. `size` chỉ định độ dài của buffer này.

Theo mặc định, các system call này tạo EA mới nếu chưa tồn tại, hoặc thay thế giá trị nếu đã tồn tại. Đối số `flags` cung cấp kiểm soát tinh hơn:
- `XATTR_CREATE`: Thất bại (`EEXIST`) nếu EA với tên đã cho đã tồn tại.
- `XATTR_REPLACE`: Thất bại (`ENODATA`) nếu EA với tên đã cho chưa tồn tại.

Ví dụ tạo user EA:

```c
char *value;
value = "The past is not dead.";
if (setxattr(pathname, "user.x", value, strlen(value), 0) == -1)
    errExit("setxattr");
```

### Lấy giá trị của EA

Các system call `getxattr()`, `lgetxattr()`, và `fgetxattr()` lấy giá trị của EA.

```c
#include <sys/xattr.h>
ssize_t getxattr(const char *pathname, const char *name, void *value,
                 size_t size);
ssize_t lgetxattr(const char *pathname, const char *name, void *value,
                  size_t size);
ssize_t fgetxattr(int fd, const char *name, void *value, size_t size);
           All return (nonnegative) size of EA value on success, or –1 on error
```

Đối số `name` xác định EA cần lấy giá trị. Giá trị EA được trả về trong buffer được trỏ bởi `value`. Buffer này phải được caller cấp phát và độ dài phải chỉ định trong `size`. Khi thành công, các system call này trả về số byte được copy vào `value`.

Nếu file không có EA với tên đã cho, lỗi `ENODATA`. Nếu `size` quá nhỏ, lỗi `ERANGE`.

Có thể chỉ định `size` là 0, trong trường hợp đó `value` bị bỏ qua nhưng system call vẫn trả về kích thước của giá trị EA — cho phép xác định kích thước buffer cần thiết cho lời gọi tiếp theo.

### Xóa EA

Các system call `removexattr()`, `lremovexattr()`, và `fremovexattr()` xóa EA khỏi file.

```c
#include <sys/xattr.h>
int removexattr(const char *pathname, const char *name);
int lremovexattr(const char *pathname, const char *name);
int fremovexattr(int fd, const char *name);
                                          All return 0 on success, or –1 on error
```

Chuỗi kết thúc null trong `name` xác định EA cần xóa. Cố xóa EA không tồn tại sẽ thất bại với lỗi `ENODATA`.

### Lấy tên của tất cả EA liên quan đến file

Các system call `listxattr()`, `llistxattr()`, và `flistxattr()` trả về danh sách chứa tên của tất cả EA liên quan đến file.

```c
#include <sys/xattr.h>
ssize_t listxattr(const char *pathname, char *list, size_t size);
ssize_t llistxattr(const char *pathname, char *list, size_t size);
ssize_t flistxattr(int fd, char *list, size_t size);
             All return number of bytes copied into list on success, or –1 on error
```

Danh sách tên EA được trả về như một loạt chuỗi kết thúc null trong buffer được trỏ bởi `list`. Kích thước buffer phải chỉ định trong `size`.

Để lấy danh sách tên EA liên quan đến file chỉ cần file có thể truy cập (quyền execute trên tất cả directory trong pathname). Không cần quyền nào trên chính file.

Vì lý do bảo mật, tên EA được trả về trong `list` có thể loại trừ các attribute mà process gọi không có quyền truy cập. Vì vậy cần xử lý khả năng lời gọi `getxattr()` tiếp theo dùng tên EA từ `list` có thể thất bại.

### Chương trình ví dụ

Chương trình trong Listing 16-1 lấy và hiển thị tên và giá trị của tất cả EA của các file được liệt kê trên command line.

```
$ setfattr -n user.x -v "The past is not dead." tfile
$ setfattr -n user.y -v "In fact, it's not even past." tfile
$ ./xattr_view tfile
tfile:
    name=user.x; value=The past is not dead.
    name=user.y; value=In fact, it's not even past.
```

**Listing 16-1:** Hiển thị extended attribute của file

```c
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– xattr/xattr_view.c
#include <sys/xattr.h>
#include "tlpi_hdr.h"
#define XATTR_SIZE 10000
static void
usageError(char *progName)
{
    fprintf(stderr, "Usage: %s [-x] file...\n", progName);
    exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
    char list[XATTR_SIZE], value[XATTR_SIZE];
    ssize_t listLen, valueLen;
    int ns, j, k, opt;
    Boolean hexDisplay;
    hexDisplay = 0;
    while ((opt = getopt(argc, argv, "x")) != -1) {
        switch (opt) {
        case 'x': hexDisplay = 1; break;
        case '?': usageError(argv[0]);
        }
    }
    if (optind >= argc + 2)
        usageError(argv[0]);
    for (j = optind; j < argc; j++) {
        listLen = listxattr(argv[j], list, XATTR_SIZE);
        if (listLen == -1)
            errExit("listxattr");
        printf("%s:\n", argv[j]);
        /* Duyệt qua tất cả tên EA, hiển thị tên + giá trị */
        for (ns = 0; ns < listLen; ns += strlen(&list[ns]) + 1) {
            printf("    name=%s; ", &list[ns]);
            valueLen = getxattr(argv[j], &list[ns], value, XATTR_SIZE);
            if (valueLen == -1) {
                printf("couldn't get value");
            } else if (!hexDisplay) {
                printf("value=%.*s", (int) valueLen, value);
            } else {
                printf("value=");
                for (k = 0; k < valueLen; k++)
                    printf("%02x ", (unsigned int) value[k]);
            }
            printf("\n");
        }
        printf("\n");
    }
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––– xattr/xattr_view.c
```

---

## 16.4 Tóm Tắt

Từ phiên bản 2.6 trở đi, Linux hỗ trợ extended attribute, cho phép gắn metadata tùy ý vào file dưới dạng các cặp tên-giá trị.

---

## 16.5 Bài Tập

**16-1.** Viết chương trình có thể dùng để tạo hoặc sửa đổi user EA cho file (tức là phiên bản đơn giản của `setfattr(1)`). Tên file và tên cùng giá trị EA phải được cung cấp làm đối số command-line.
