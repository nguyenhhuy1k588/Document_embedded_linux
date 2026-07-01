## Chương 17
# Access Control List (ACL)

Mục 15.4 đã mô tả scheme quyền truy cập file UNIX (và Linux) truyền thống. Đối với nhiều ứng dụng, scheme này là đủ. Tuy nhiên, một số ứng dụng cần kiểm soát tinh hơn về quyền được cấp cho các user và group cụ thể. Để đáp ứng yêu cầu này, nhiều hệ thống UNIX triển khai một phần mở rộng cho model quyền file UNIX truyền thống gọi là access control list (ACL). ACL cho phép quyền file được chỉ định theo từng user hoặc từng group, cho số lượng user và group tùy ý. Linux cung cấp ACL từ kernel 2.6 trở đi.

> Hỗ trợ ACL là tùy chọn cho mỗi file system. Để có thể tạo ACL trên ext2, ext3, ext4, hoặc Reiserfs, file system phải được mount với tùy chọn `mount -o acl`.

ACL chưa bao giờ được chuẩn hóa chính thức cho các hệ thống UNIX. Nhiều triển khai UNIX (gồm Linux) dựa trên các draft chuẩn POSIX.1e và POSIX.2c (thường là phiên bản cuối, Draft 17).

---

## 17.1 Tổng Quan

ACL là một loạt các ACL entry, mỗi entry định nghĩa quyền file cho một user riêng lẻ hoặc một group của user.

```
┌──────────────┬───────────────┬─────────────┐
│  Tag type    │ Tag qualifier │ Permissions │
├──────────────┼───────────────┼─────────────┤
│ ACL_USER_OBJ │       -       │     rwx     │ ← Tương ứng quyền owner truyền thống
│ ACL_USER     │     1007      │     r--     │
│ ACL_USER     │     1010      │     rwx     │
├──────────────┼───────────────┼─────────────┤
│ACL_GROUP_OBJ │       -       │     rwx     │ ← Tương ứng quyền group truyền thống
│ ACL_GROUP    │      102      │     r--     │
│ ACL_GROUP    │      103      │     -w-     │
│ ACL_GROUP    │      109      │     --x     │
├──────────────┼───────────────┼─────────────┤
│   ACL_MASK   │       -       │     rw-     │
├──────────────┼───────────────┼─────────────┤
│  ACL_OTHER   │       -       │     r--     │ ← Tương ứng quyền other truyền thống
└──────────────┴───────────────┴─────────────┘
  ↑ Group class entry: ACL_USER, ACL_GROUP_OBJ, ACL_GROUP
```

**Hình 17-1:** Access control list

### ACL entry

Mỗi ACL entry gồm các phần sau:

- **Tag type**: Chỉ ra entry này áp dụng cho user, group, hay loại user nào khác;
- **Tag qualifier** (tùy chọn): Xác định user hoặc group cụ thể (tức là user ID hoặc group ID); và
- **Permission set**: Chỉ định các quyền (read, write, và execute) được cấp bởi entry.

Tag type có một trong các giá trị sau:

#### ACL_USER_OBJ
Entry này chỉ định quyền được cấp cho file owner. Mỗi ACL chứa đúng một `ACL_USER_OBJ` entry. Entry này tương ứng với quyền owner (user) file truyền thống.

#### ACL_USER
Entry này chỉ định quyền được cấp cho user được xác định bởi tag qualifier. Một ACL có thể chứa không hoặc nhiều `ACL_USER` entry, nhưng tối đa một entry cho mỗi user.

#### ACL_GROUP_OBJ
Entry này chỉ định quyền được cấp cho file group. Mỗi ACL chứa đúng một `ACL_GROUP_OBJ` entry. Entry này tương ứng với quyền group file truyền thống, trừ khi ACL cũng chứa `ACL_MASK` entry.

#### ACL_GROUP
Entry này chỉ định quyền được cấp cho group được xác định bởi tag qualifier. Một ACL có thể chứa không hoặc nhiều `ACL_GROUP` entry, nhưng tối đa một entry cho mỗi group.

#### ACL_MASK
Entry này chỉ định quyền tối đa có thể được cấp bởi các entry `ACL_USER`, `ACL_GROUP_OBJ`, và `ACL_GROUP`. Một ACL chứa tối đa một `ACL_MASK` entry. Nếu ACL chứa `ACL_USER` hoặc `ACL_GROUP` entry, thì `ACL_MASK` entry là bắt buộc.

#### ACL_OTHER
Entry này chỉ định quyền được cấp cho user không khớp với bất kỳ ACL entry nào khác. Mỗi ACL chứa đúng một `ACL_OTHER` entry. Tương ứng với quyền other truyền thống.

### ACL tối thiểu và ACL mở rộng

**Minimal ACL** là ACL tương đương ngữ nghĩa với tập quyền file truyền thống. Nó chứa đúng ba entry: một của mỗi loại `ACL_USER_OBJ`, `ACL_GROUP_OBJ`, và `ACL_OTHER`.

**Extended ACL** là ACL bổ sung chứa các entry `ACL_USER`, `ACL_GROUP`, và `ACL_MASK`.

ACL được triển khai dưới dạng system extended attribute (Chương 16). Extended attribute dùng để lưu access ACL có tên là `system.posix_acl_access`. Extended attribute này chỉ cần thiết nếu file có extended ACL. Thông tin quyền cho minimal ACL được lưu trong các bit quyền file truyền thống.

---

## 17.2 Thuật Toán Kiểm Tra Quyền ACL

Kiểm tra quyền trên file có ACL được thực hiện trong các trường hợp tương tự như model quyền file truyền thống. Kiểm tra được thực hiện theo thứ tự sau, cho đến khi một trong các tiêu chí được khớp:

1. Nếu process có đặc quyền, tất cả truy cập được cấp. Ngoại lệ: khi thực thi file, process có đặc quyền chỉ được cấp quyền execute nếu quyền đó được cấp qua ít nhất một trong các ACL entry.
2. Nếu effective user ID của process khớp với owner (user ID) của file, process được cấp quyền được chỉ định trong `ACL_USER_OBJ` entry.
3. Nếu effective user ID của process khớp với tag qualifier trong một trong các `ACL_USER` entry, process được cấp quyền được chỉ định trong entry đó, được mask (AND) với giá trị của `ACL_MASK` entry.
4. Nếu một trong các group ID của process (effective group ID hoặc bất kỳ supplementary group ID nào) khớp với file group hoặc tag qualifier của bất kỳ `ACL_GROUP` entry nào:
   - a) Nếu group ID khớp với file group và `ACL_GROUP_OBJ` entry cấp quyền yêu cầu, entry này xác định quyền. Quyền được mask với `ACL_MASK` entry nếu có.
   - b) Nếu group ID khớp với tag qualifier trong `ACL_GROUP` entry và entry đó cấp quyền yêu cầu, entry này xác định quyền, được mask với `ACL_MASK` entry.
   - c) Nếu không, quyền truy cập bị từ chối.
5. Nếu không, process được cấp quyền được chỉ định trong `ACL_OTHER` entry.

---

## 17.3 Dạng Văn Bản Dài và Ngắn cho ACL

Khi thao tác ACL bằng lệnh `setfacl` và `getfacl` hoặc một số hàm thư viện ACL, ta chỉ định biểu diễn dạng văn bản của các ACL entry. Hai định dạng được phép:

- **Long text form**: Chứa một ACL entry mỗi dòng, có thể bao gồm comment (bắt đầu bằng `#`). Lệnh `getfacl` hiển thị ACL theo dạng này.
- **Short text form**: Bao gồm chuỗi ACL entry phân cách bằng dấu phẩy.

Trong cả hai dạng, mỗi ACL entry gồm ba phần phân cách bằng dấu hai chấm:
```
tag-type:[tag-qualifier]:permissions
```

Ví dụ short text form tương ứng với permission mask `0650`:
```
u::rw-,g::r-x,o::---
```

ACL ngắn có hai named user, một named group, và mask entry:
```
u::rw,u:paulh:rw,u:annabel:rw,g::r,g:teach:rw,m::rwx,o::-
```

**Bảng 17-1:** Diễn giải dạng văn bản ACL entry

| Dạng tag text | Tag qualifier? | Loại tag tương ứng | Entry cho |
|--------------|----------------|---------------------|-----------|
| u, user | Không | `ACL_USER_OBJ` | File owner (user) |
| u, user | Có | `ACL_USER` | User được chỉ định |
| g, group | Không | `ACL_GROUP_OBJ` | File group |
| g, group | Có | `ACL_GROUP` | Group được chỉ định |
| m, mask | Không | `ACL_MASK` | Mask cho group class |
| o, other | Không | `ACL_OTHER` | Các user khác |

---

## 17.4 ACL_MASK Entry và ACL Group Class

Nếu ACL chứa `ACL_USER` hoặc `ACL_GROUP` entry, thì phải có `ACL_MASK` entry.

`ACL_MASK` entry hoạt động như giới hạn trên cho quyền được cấp bởi các ACL entry trong group class. Group class là tập tất cả các `ACL_USER`, `ACL_GROUP`, và `ACL_GROUP_OBJ` entry trong ACL.

Mục đích của `ACL_MASK` entry là cung cấp hành vi nhất quán khi chạy các ứng dụng không biết về ACL. Khi ACL có `ACL_MASK` entry:

- Tất cả thay đổi đối với quyền group truyền thống qua `chmod()` thay đổi cài đặt của `ACL_MASK` entry (thay vì `ACL_GROUP_OBJ` entry).
- Lời gọi `stat()` trả về quyền `ACL_MASK` (thay vì quyền `ACL_GROUP_OBJ`) trong các group permission bit của trường `st_mode`.

---

## 17.5 Lệnh getfacl và setfacl

Từ shell, dùng `getfacl` để xem ACL trên file:

```
$ umask 022
$ touch tfile
$ getfacl tfile
# file: tfile
# owner: mtk
# group: users
user::rw-
group::r--
other::r--
```

File mới được tạo với minimal ACL. Thay đổi quyền bằng `chmod` được phản ánh trong ACL:

```
$ chmod u=rwx,g=rx,o=x tfile
$ getfacl --omit-header tfile
user::rwx
group::r-x
other::--x
```

Dùng `setfacl -m` để thêm `ACL_USER` và `ACL_GROUP` entry:

```
$ setfacl -m u:paulh:rx,g:teach:x tfile
$ getfacl --omit-header tfile
user::rwx
user:paulh:r-x
group::r-x
group:teach:--x
mask::r-x          ← setfacl tự động tạo ACL_MASK entry
other::--x
```

`setfacl` tự động tạo `ACL_MASK` entry. Việc thêm entry chuyển ACL thành extended ACL và `ls -l` chỉ ra điều này bằng dấu `+` sau permission mask:

```
$ ls -l tfile
-rwxr-x--x+ 1 mtk users 0 Dec  3 15:42 tfile
```

Dùng `setfacl -x` để xóa entry:

```
$ setfacl -x u:paulh,g:teach tfile
```

`setfacl -b` xóa tất cả extended entry, giữ lại chỉ minimal entry.

---

## 17.6 Default ACL và Tạo File

Ngoài **access ACL** (dùng để xác định quyền khi process truy cập file), ta có thể tạo **default ACL** trên directory.

Default ACL không tham gia xác định quyền khi truy cập directory. Thay vào đó, sự hiện diện hay vắng mặt của nó xác định ACL và quyền được đặt trên các file và subdirectory được tạo trong directory. (Default ACL được lưu dưới dạng extended attribute có tên `system.posix_acl_default`.)

Dùng tùy chọn `-d` của `getfacl` và `setfacl` để xem và đặt default ACL:

```
$ mkdir sub
$ setfacl -d -m u::rwx,u:paulh:rx,g::rx,g:teach:rwx,o::- sub
$ getfacl -d --omit-header sub
user::rwx
user:paulh:r-x
group::r-x
group:teach:rwx
mask::rwx
other::---
```

Dùng `setfacl -k` để xóa default ACL khỏi directory.

Nếu directory có default ACL:

- Subdirectory mới được tạo trong directory này kế thừa default ACL của directory như default ACL của nó.
- File hoặc subdirectory mới được tạo trong directory này kế thừa default ACL của directory như access ACL của nó. Các entry ACL tương ứng với bit quyền file truyền thống được mask (AND) với các bit tương ứng của đối số `mode` trong system call (`open()`, `mkdir()`, ...) được dùng để tạo file.
- Umask của process không tham gia xác định quyền khi directory có default ACL.

Nếu directory không có default ACL, các file và directory con mới cũng không có default ACL, và quyền được đặt theo quy tắc truyền thống (Mục 15.4.6).

---

## 17.7 Giới Hạn Triển Khai ACL

Các triển khai file system khác nhau áp đặt giới hạn về số entry trong ACL:

- **ext2, ext3, ext4**: Tổng số ACL trên file bị chi phối bởi yêu cầu rằng byte trong tất cả tên và giá trị của EA phải nằm trong một logical disk block. Mỗi ACL entry cần 8 byte, nên block size 4096 byte cho phép tối đa khoảng 500 ACL entry.
- **XFS**: ACL bị giới hạn ở 25 entry.
- **Reiserfs và JFS**: ACL có thể chứa đến 8191 entry.

Mặc dù hầu hết file system cho phép số lượng lớn entry, nên tránh điều này vì: quản lý ACL dài trở nên phức tạp, và thời gian quét ACL để tìm entry khớp tăng tuyến tính với số entry.

---

## 17.8 ACL API

Draft chuẩn POSIX.1e định nghĩa nhiều hàm và cấu trúc dữ liệu để thao tác ACL. Chương trình dùng ACL API phải include `<sys/acl.h>` (và có thể `<acl/libacl.h>` cho các Linux extension), và biên dịch với tùy chọn `-lacl`.

ACL API coi ACL là đối tượng phân cấp:
- ACL gồm một hoặc nhiều ACL entry.
- Mỗi ACL entry gồm tag type, tag qualifier tùy chọn, và permission set.

### Lấy ACL của file vào memory

```c
acl_t acl;
acl = acl_get_file(pathname, type);
```

`type` là `ACL_TYPE_ACCESS` (access ACL) hoặc `ACL_TYPE_DEFAULT` (default ACL). Trả về handle kiểu `acl_t`.

### Lấy entry từ in-memory ACL

```c
acl_entry_t entry;
status = acl_get_entry(acl, entry_id, &entry);
```

Nếu `entry_id` là `ACL_FIRST_ENTRY`: trả về handle của entry đầu tiên. Nếu là `ACL_NEXT_ENTRY`: trả về handle của entry tiếp theo. Trả về 1 nếu thành công, 0 nếu không còn entry, -1 khi lỗi.

### Lấy và sửa thuộc tính trong ACL entry

```c
acl_tag_t tag_type;
status = acl_get_tag_type(entry, &tag_type);
status = acl_set_tag_type(entry, tag_type);

uid_t *qualp;
qualp = acl_get_qualifier(entry);
status = acl_set_qualifier(entry, qualp);

acl_permset_t permset;
status = acl_get_permset(entry, &permset);
status = acl_set_permset(entry, permset);
```

Các hàm thao tác permission set:

```c
int is_set;
is_set = acl_get_perm(permset, perm);   /* Kiểm tra: 1 = có, 0 = không */
status = acl_add_perm(permset, perm);   /* Thêm quyền */
status = acl_delete_perm(permset, perm); /* Xóa quyền */
status = acl_clear_perms(permset);      /* Xóa tất cả quyền */
```

`perm` là `ACL_READ`, `ACL_WRITE`, hoặc `ACL_EXECUTE`.

### Tạo và xóa ACL entry

```c
acl_entry_t entry;
status = acl_create_entry(&acl, &entry);  /* Tạo entry mới */
status = acl_delete_entry(acl, entry);    /* Xóa entry */
```

### Cập nhật ACL của file

```c
int status;
status = acl_set_file(pathname, type, acl);
```

### Chuyển đổi ACL giữa in-memory và text form

```c
acl = acl_from_text(acl_string);   /* Text → in-memory */

char *str;
ssize_t len;
str = acl_to_text(acl, &len);      /* in-memory → text */
```

### Các hàm ACL khác

- `acl_calc_mask(&acl)`: Tính và đặt quyền trong `ACL_MASK` entry (union của tất cả `ACL_USER`, `ACL_GROUP`, và `ACL_GROUP_OBJ` entry). Tạo `ACL_MASK` entry nếu chưa có.
- `acl_valid(acl)`: Trả về 0 nếu ACL hợp lệ, -1 nếu không.
- `acl_delete_def_file(pathname)`: Xóa default ACL trên directory.
- `acl_init(count)`: Tạo ACL trống mới.
- `acl_dup(acl)`: Tạo bản sao của ACL.
- `acl_free(handle)`: Giải phóng bộ nhớ được cấp phát bởi các hàm ACL khác.

### Chương trình ví dụ

Listing 17-1 dùng một số hàm thư viện ACL để lấy và hiển thị ACL trên file.

```
$ touch tfile
$ setfacl -m 'u:annie:r,u:paulh:rw,g:teach:r' tfile
$ ./acl_view tfile
user_obj     rw-
user annie   r--
user paulh   rw-
group_obj    r--
group teach  r--
mask         rw-
other        r--
```

**Listing 17-1:** Hiển thị access hoặc default ACL trên file

```c
––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– acl/acl_view.c
#include <acl/libacl.h>
#include <sys/acl.h>
#include "ugid_functions.h"
#include "tlpi_hdr.h"
static void
usageError(char *progName)
{
    fprintf(stderr, "Usage: %s [-d] filename\n", progName);
    exit(EXIT_FAILURE);
}
int
main(int argc, char *argv[])
{
    acl_t acl;
    acl_type_t type;
    acl_entry_t entry;
    acl_tag_t tag;
    uid_t *uidp;
    gid_t *gidp;
    acl_permset_t permset;
    char *name;
    int entryId, permVal, opt;
    type = ACL_TYPE_ACCESS;
    while ((opt = getopt(argc, argv, "d")) != -1) {
        switch (opt) {
        case 'd': type = ACL_TYPE_DEFAULT; break;
        case '?': usageError(argv[0]);
        }
    }
    if (optind + 1 != argc)
        usageError(argv[0]);
    acl = acl_get_file(argv[optind], type);
    if (acl == NULL)
        errExit("acl_get_file");
    /* Duyệt qua từng entry trong ACL này */
    for (entryId = ACL_FIRST_ENTRY; ; entryId = ACL_NEXT_ENTRY) {
        if (acl_get_entry(acl, entryId, &entry) != 1)
            break;
        /* Lấy và hiển thị tag type */
        if (acl_get_tag_type(entry, &tag) == -1)
            errExit("acl_get_tag_type");
        printf("%-12s", (tag == ACL_USER_OBJ)  ? "user_obj"  :
                        (tag == ACL_USER)       ? "user"      :
                        (tag == ACL_GROUP_OBJ)  ? "group_obj" :
                        (tag == ACL_GROUP)      ? "group"     :
                        (tag == ACL_MASK)       ? "mask"      :
                        (tag == ACL_OTHER)      ? "other"     : "???");
        /* Lấy và hiển thị tag qualifier nếu có */
        if (tag == ACL_USER) {
            uidp = acl_get_qualifier(entry);
            if (uidp == NULL)  errExit("acl_get_qualifier");
            name = groupNameFromId(*uidp);
            if (name == NULL)  printf("%-8d ", *uidp);
            else               printf("%-8s ", name);
            if (acl_free(uidp) == -1) errExit("acl_free");
        } else if (tag == ACL_GROUP) {
            gidp = acl_get_qualifier(entry);
            if (gidp == NULL)  errExit("acl_get_qualifier");
            name = groupNameFromId(*gidp);
            if (name == NULL)  printf("%-8d ", *gidp);
            else               printf("%-8s ", name);
            if (acl_free(gidp) == -1) errExit("acl_free");
        } else {
            printf("         ");
        }
        /* Lấy và hiển thị quyền */
        if (acl_get_permset(entry, &permset) == -1)
            errExit("acl_get_permset");
        permVal = acl_get_perm(permset, ACL_READ);
        if (permVal == -1)  errExit("acl_get_perm - ACL_READ");
        printf("%c", (permVal == 1) ? 'r' : '-');
        permVal = acl_get_perm(permset, ACL_WRITE);
        if (permVal == -1)  errExit("acl_get_perm - ACL_WRITE");
        printf("%c", (permVal == 1) ? 'w' : '-');
        permVal = acl_get_perm(permset, ACL_EXECUTE);
        if (permVal == -1)  errExit("acl_get_perm - ACL_EXECUTE");
        printf("%c\n", (permVal == 1) ? 'x' : '-');
    }
    if (acl_free(acl) == -1)
        errExit("acl_free");
    exit(EXIT_SUCCESS);
}
–––––––––––––––––––––––––––––––––––––––––––––––––––––––––––– acl/acl_view.c
```

---

## 17.9 Tóm Tắt

Từ phiên bản 2.6 trở đi, Linux hỗ trợ ACL. ACL mở rộng model quyền file UNIX truyền thống, cho phép kiểm soát quyền file theo từng user và từng group.

---

## 17.10 Bài Tập

**17-1.** Viết chương trình hiển thị quyền từ ACL entry tương ứng với một user hoặc group cụ thể. Chương trình nhận hai đối số command-line. Đối số đầu tiên là chữ `u` hoặc `g`, chỉ ra đối số thứ hai xác định user hay group. Nếu ACL entry rơi vào group class, chương trình cũng phải hiển thị quyền sẽ áp dụng sau khi ACL entry được mask bởi ACL mask entry.
