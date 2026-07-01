## Chương 8
# Người Dùng và Nhóm (Users and Groups)

Mỗi người dùng có một tên đăng nhập duy nhất và một số định danh người dùng (UID) đi kèm. Người dùng có thể thuộc một hoặc nhiều group. Mỗi group cũng có tên duy nhất và một số định danh group (GID).

Mục đích chính của user ID và group ID là xác định quyền sở hữu các tài nguyên hệ thống và kiểm soát quyền truy cập của process vào các tài nguyên đó. Ví dụ, mỗi file thuộc về một user và group nhất định, và mỗi process có một số user ID và group ID xác định ai sở hữu process đó và process có những quyền gì khi truy cập file (xem Chương 9 để biết thêm chi tiết).

Trong chương này, chúng ta xem xét các file hệ thống dùng để định nghĩa user và group, sau đó mô tả các hàm thư viện dùng để lấy thông tin từ các file đó. Cuối chương là thảo luận về hàm `crypt()`, dùng để mã hóa và xác thực mật khẩu đăng nhập.

---

## 8.1 File Password: /etc/passwd

File password hệ thống, `/etc/passwd`, chứa một dòng cho mỗi tài khoản người dùng trên hệ thống. Mỗi dòng gồm bảy trường phân cách bằng dấu hai chấm (`:`), ví dụ:

```
mtk:x:1000:100:Michael Kerrisk:/home/mtk:/bin/bash
```

Lần lượt các trường đó như sau:

- **Login name**: Tên duy nhất mà người dùng phải nhập khi đăng nhập. Thường còn gọi là username. Đây là định danh dạng người đọc được (symbolic) tương ứng với numeric user ID. Các chương trình như `ls(1)` hiển thị tên này, thay vì numeric user ID, khi cần hiển thị quyền sở hữu file (ví dụ `ls -l`).

- **Encrypted password**: Trường này chứa mật khẩu đã mã hóa dài 13 ký tự, mô tả chi tiết hơn ở Mục 8.5. Nếu trường này chứa chuỗi khác — đặc biệt là chuỗi không phải 13 ký tự — thì đăng nhập vào tài khoản đó bị vô hiệu hóa, vì chuỗi như vậy không biểu diễn mật khẩu mã hóa hợp lệ. Tuy nhiên, trường này bị bỏ qua nếu shadow password đã được bật (thường là như vậy). Trong trường hợp đó, trường password trong `/etc/passwd` theo quy ước chứa chữ `x` (dù bất kỳ chuỗi khác không rỗng cũng được), và mật khẩu mã hóa được lưu trong shadow password file (Mục 8.2). Nếu trường password trong `/etc/passwd` rỗng thì không cần mật khẩu để đăng nhập (đúng ngay cả khi shadow password đã bật).

  Ở đây, chúng ta giả định mật khẩu được mã hóa bằng Data Encryption Standard (DES) — thuật toán mã hóa mật khẩu UNIX lịch sử và vẫn được sử dụng rộng rãi. Có thể thay DES bằng các thuật toán khác như MD5, tạo ra message digest 128-bit của đầu vào, lưu dưới dạng chuỗi 34 ký tự trong file password (hoặc shadow password).

- **User ID (UID)**: Numeric ID của người dùng. Nếu trường này bằng 0, tài khoản đó có đặc quyền superuser. Thông thường chỉ có một tài khoản như vậy, tên đăng nhập là `root`. Trên Linux 2.2 và trước đó, user ID lưu dưới dạng 16-bit (0–65535); trên Linux 2.4 trở đi, lưu bằng 32-bit.

  > Có thể (nhưng hiếm) có nhiều bản ghi trong file password cùng user ID, cho phép nhiều tên đăng nhập cho cùng một user ID. Điều này cho phép nhiều người dùng truy cập cùng tài nguyên (như file) với các mật khẩu khác nhau. Các tên đăng nhập khác nhau có thể liên kết với các tập group ID khác nhau.

- **Group ID (GID)**: Numeric ID của group đầu tiên mà người dùng thuộc về. Các group membership bổ sung được định nghĩa trong group file của hệ thống.

- **Comment**: Trường này chứa văn bản mô tả về người dùng. Nội dung này được hiển thị bởi các chương trình như `finger(1)`.

- **Home directory**: Thư mục mà người dùng được đặt vào sau khi đăng nhập. Trường này trở thành giá trị của biến môi trường `HOME`.

- **Login shell**: Chương trình được chuyển quyền điều khiển sau khi người dùng đăng nhập. Thường là một shell như `bash`, nhưng có thể là bất kỳ chương trình nào. Nếu trường này rỗng thì login shell mặc định là `/bin/sh` (Bourne shell). Trường này trở thành giá trị của biến môi trường `SHELL`.

Trên hệ thống độc lập, toàn bộ thông tin password nằm trong `/etc/passwd`. Tuy nhiên, nếu dùng hệ thống như Network Information System (NIS) hoặc Lightweight Directory Access Protocol (LDAP) để phân phối mật khẩu trong môi trường mạng, một phần hoặc toàn bộ thông tin đó nằm trên hệ thống từ xa. Miễn là các chương trình truy cập thông tin password dùng các hàm mô tả trong chương này (`getpwnam()`, `getpwuid()`, ...), việc dùng NIS hay LDAP là trong suốt với ứng dụng. Nhận xét tương tự áp dụng cho shadow password file và group file ở các mục sau.

---

## 8.2 Shadow Password File: /etc/shadow

Về lịch sử, các hệ thống UNIX lưu toàn bộ thông tin người dùng, kể cả mật khẩu mã hóa, trong `/etc/passwd`. Điều này tạo ra vấn đề bảo mật. Vì nhiều tiện ích hệ thống không có đặc quyền cần đọc các thông tin khác trong file password, file phải được đặt quyền đọc cho tất cả người dùng. Điều này tạo điều kiện cho các chương trình crack mật khẩu, cố gắng mã hóa các danh sách mật khẩu có thể (ví dụ: từ trong từ điển hay tên người) để xem có khớp với mật khẩu mã hóa của người dùng nào không. Shadow password file, `/etc/shadow`, được đưa ra như một biện pháp ngăn chặn các cuộc tấn công này. Ý tưởng là: tất cả thông tin người dùng không nhạy cảm nằm trong file password có thể đọc công khai, còn mật khẩu mã hóa được lưu trong shadow password file, chỉ có các chương trình có đặc quyền mới đọc được.

Ngoài tên đăng nhập (để khớp với bản ghi tương ứng trong file password) và mật khẩu mã hóa, shadow password file còn chứa một số trường liên quan đến bảo mật khác. Chi tiết các trường này có trong manual page `shadow(5)`. Chúng ta chủ yếu quan tâm đến trường mật khẩu mã hóa, được thảo luận kỹ hơn trong Mục 8.5.

SUSv3 không quy định shadow password, và không phải tất cả các triển khai UNIX đều cung cấp tính năng này.

---

## 8.3 Group File: /etc/group

Để phục vụ các mục đích quản trị — đặc biệt là kiểm soát truy cập vào file và các tài nguyên hệ thống khác — việc tổ chức người dùng thành các group là rất hữu ích.

Tập các group mà người dùng thuộc về được xác định bởi sự kết hợp của trường group ID trong bản ghi password của người dùng và các group mà người dùng được liệt kê trong group file. Cách chia thông tin kỳ lạ qua hai file này có nguồn gốc lịch sử. Trong các triển khai UNIX sớm, người dùng chỉ có thể thuộc một group tại một thời điểm. Group membership ban đầu khi đăng nhập được xác định bởi trường group ID của file password và có thể thay đổi sau đó bằng lệnh `newgrp(1)`, yêu cầu người dùng cung cấp mật khẩu group (nếu group có bảo vệ mật khẩu). 4.2BSD giới thiệu khái niệm multiple simultaneous group memberships, được chuẩn hóa sau đó trong POSIX.1-1990. Theo scheme này, group file liệt kê các group membership bổ sung của mỗi người dùng. (Lệnh `groups(1)` hiển thị các group mà shell process là thành viên, hoặc nếu có tên người dùng làm đối số thì hiển thị group membership của những người dùng đó.)

Group file, `/etc/group`, chứa một dòng cho mỗi group trong hệ thống. Mỗi dòng gồm bốn trường phân cách bằng dấu hai chấm, ví dụ:

```
users:x:100:
jambit:x:106:claus,felli,frank,harti,markus,martin,mtk,paul
```

Lần lượt các trường đó như sau:

- **Group name**: Tên của group. Tương tự tên đăng nhập trong file password, đây là định danh dạng người đọc được (symbolic) tương ứng với numeric group identifier.

- **Encrypted password**: Trường này chứa mật khẩu tùy chọn cho group. Kể từ khi có multiple group memberships, mật khẩu group hiếm khi được dùng trên các hệ thống UNIX. Tuy nhiên, có thể đặt mật khẩu cho group (người dùng có đặc quyền có thể làm điều này bằng lệnh `passwd`). Nếu người dùng không phải thành viên group, `newgrp(1)` sẽ yêu cầu mật khẩu này trước khi khởi động shell mới với group membership gồm group đó. Nếu password shadowing được bật, trường này bị bỏ qua (theo quy ước chứa chữ `x`, nhưng chuỗi bất kỳ, kể cả rỗng, đều có thể) và mật khẩu mã hóa được lưu trong shadow group file `/etc/gshadow`, chỉ truy cập được bởi các chương trình và người dùng có đặc quyền. Mật khẩu group được mã hóa tương tự mật khẩu người dùng (Mục 8.5).

- **Group ID (GID)**: Numeric ID của group. Thông thường có một group được định nghĩa với group ID 0, tên là `root` (giống bản ghi `/etc/passwd` với user ID 0). Trên Linux 2.2 và trước đó, group ID lưu dưới dạng 16-bit (0–65535); trên Linux 2.4 trở đi, lưu bằng 32-bit.

- **User list**: Danh sách tên người dùng (phân cách bằng dấu phẩy) là thành viên của group này. (Danh sách dùng username chứ không phải user ID, vì như đã nói, user ID không nhất thiết là duy nhất trong file password.)

Để ghi nhận rằng người dùng `avr` là thành viên của các group `users`, `staff`, và `teach`, ta sẽ thấy bản ghi sau trong file password:

```
avr:x:1001:100:Anthony Robins:/home/avr:/bin/bash
```

Và các bản ghi sau trong group file:

```
users:x:100:
staff:x:101:mtk,avr,martinl
teach:x:104:avr,rlb,alc
```

Trường thứ tư của bản ghi password, chứa group ID 100, chỉ định membership của group `users`. Các group membership còn lại được chỉ định bằng cách liệt kê `avr` một lần trong từng bản ghi liên quan trong group file.

---

## 8.4 Lấy Thông Tin User và Group

Trong mục này, chúng ta xem xét các hàm thư viện cho phép lấy từng bản ghi từ file password, shadow password, và group file, cũng như để duyệt qua toàn bộ bản ghi trong mỗi file.

### Lấy bản ghi từ file password

Các hàm `getpwnam()` và `getpwuid()` lấy bản ghi từ file password.

```c
#include <pwd.h>
struct passwd *getpwnam(const char *name);
struct passwd *getpwuid(uid_t uid);
                              Cả hai trả về pointer khi thành công, hoặc NULL khi lỗi;
                            xem phần chính để biết trường hợp "not found"
```

Với tên đăng nhập trong `name`, hàm `getpwnam()` trả về pointer đến một structure có kiểu sau, chứa thông tin tương ứng từ bản ghi password:

```c
struct passwd {
    char *pw_name;   /* Login name (username) */
    char *pw_passwd; /* Encrypted password */
    uid_t pw_uid;    /* User ID */
    gid_t pw_gid;    /* Group ID */
    char *pw_gecos;  /* Comment (user information) */
    char *pw_dir;    /* Initial working (home) directory */
    char *pw_shell;  /* Login shell */
};
```

Các trường `pw_gecos` và `pw_passwd` của structure `passwd` không được định nghĩa trong SUSv3, nhưng có sẵn trên tất cả các triển khai UNIX. Trường `pw_passwd` chứa thông tin hợp lệ chỉ khi shadow password không được bật. (Về mặt lập trình, cách đơn giản nhất để xác định shadow password có được bật không là gọi `getpwnam()` thành công rồi gọi tiếp `getspnam()`, mô tả ngay sau, để xem nó có trả về bản ghi shadow password cho cùng username không.) Một số triển khai khác cung cấp thêm các trường không chuẩn trong structure này.

> Trường `pw_gecos` lấy tên từ các triển khai UNIX sớm, nơi trường này chứa thông tin dùng để giao tiếp với máy chạy General Electric Comprehensive Operating System (GECOS). Dù mục đích đó đã lỗi thời từ lâu, tên trường vẫn còn tồn tại và được dùng để lưu thông tin về người dùng.

Hàm `getpwuid()` trả về đúng thông tin như `getpwnam()`, nhưng tra cứu bằng numeric user ID được cung cấp trong đối số `uid`.

Cả `getpwnam()` và `getpwuid()` đều trả về pointer đến một structure được cấp phát tĩnh. Structure này bị ghi đè mỗi khi gọi một trong hai hàm này (hoặc hàm `getpwent()` mô tả bên dưới).

Vì trả về pointer đến bộ nhớ được cấp phát tĩnh, `getpwnam()` và `getpwuid()` không phải là reentrant. Thực tế còn phức tạp hơn, vì structure `passwd` trả về chứa các pointer đến thông tin khác (ví dụ: trường `pw_name`) cũng được cấp phát tĩnh. (Chúng ta giải thích reentrancy trong Mục 21.1.2.) Nhận xét tương tự áp dụng cho các hàm `getgrnam()` và `getgrgid()` (mô tả ngay sau).

SUSv3 quy định một tập hàm reentrant tương đương — `getpwnam_r()`, `getpwuid_r()`, `getgrnam_r()`, và `getgrgid_r()` — nhận thêm đối số là cả structure `passwd` (hoặc `group`) lẫn vùng buffer để giữ các structure khác mà các trường của `passwd` (`group`) trỏ đến. Số byte cần thiết cho buffer bổ sung này có thể lấy bằng `sysconf(_SC_GETPW_R_SIZE_MAX)` (hoặc `sysconf(_SC_GETGR_R_SIZE_MAX)` cho các hàm liên quan group). Xem manual page để biết chi tiết.

Theo SUSv3, nếu không tìm thấy bản ghi `passwd` khớp, thì `getpwnam()` và `getpwuid()` phải trả về `NULL` và để `errno` không thay đổi. Điều này có nghĩa ta có thể phân biệt trường hợp lỗi và "not found" bằng code như sau:

```c
struct passwd *pwd;
errno = 0;
pwd = getpwnam(name);
if (pwd == NULL) {
    if (errno == 0)
        /* Not found */;
    else
        /* Error */;
}
```

Tuy nhiên, nhiều triển khai UNIX không tuân theo SUSv3 ở điểm này. Nếu không tìm thấy bản ghi `passwd` khớp, các hàm đó trả về `NULL` và đặt `errno` về giá trị khác 0 như `ENOENT` hoặc `ESRCH`. Trước phiên bản 2.7, glibc sinh ra lỗi `ENOENT` cho trường hợp này, nhưng từ phiên bản 2.7, glibc tuân theo yêu cầu SUSv3. Sự biến thể giữa các triển khai xuất phát một phần từ việc POSIX.1-1990 không yêu cầu các hàm này đặt `errno` khi lỗi và cho phép đặt `errno` cho trường hợp "not found". Hệ quả là thực sự không thể phân biệt một cách portable trường hợp lỗi và "not found" khi dùng các hàm này.

### Lấy bản ghi từ group file

Các hàm `getgrnam()` và `getgrgid()` lấy bản ghi từ group file.

```c
#include <grp.h>
struct group *getgrnam(const char *name);
struct group *getgrgid(gid_t gid);
                                Cả hai trả về pointer khi thành công, hoặc NULL khi lỗi;
                            xem phần chính để biết trường hợp "not found"
```

Hàm `getgrnam()` tra cứu thông tin group theo tên group, còn `getgrgid()` tra cứu theo group ID. Cả hai hàm trả về pointer đến structure có kiểu sau:

```c
struct group {
    char  *gr_name;   /* Group name */
    char  *gr_passwd; /* Encrypted password (if not password shadowing) */
    gid_t  gr_gid;    /* Group ID */
    char **gr_mem;    /* NULL-terminated array of pointers to names
                         of members listed in /etc/group */
};
```

Trường `gr_passwd` của structure `group` không được quy định trong SUSv3, nhưng có sẵn trên hầu hết các triển khai UNIX.

Giống như các hàm password tương ứng mô tả ở trên, structure này bị ghi đè mỗi khi gọi một trong các hàm này.

Nếu các hàm không tìm thấy bản ghi group khớp, chúng thể hiện hành vi biến thể tương tự như đã mô tả cho `getpwnam()` và `getpwuid()`.

### Chương trình ví dụ

Một trong những cách dùng phổ biến của các hàm đã mô tả là chuyển đổi tên người dùng và group dạng symbolic sang numeric ID và ngược lại. Listing 8-1 minh họa các phép chuyển đổi này dưới dạng bốn hàm: `userNameFromId()`, `userIdFromName()`, `groupNameFromId()`, và `groupIdFromName()`. Để tiện cho người gọi, `userIdFromName()` và `groupIdFromName()` cũng chấp nhận đối số `name` là chuỗi số thuần túy; trong trường hợp đó, chuỗi được chuyển đổi trực tiếp thành số và trả về cho người gọi. Chúng ta dùng các hàm này trong một số chương trình ví dụ về sau trong sách.

**Listing 8-1:** Các hàm chuyển đổi user và group ID sang/từ tên user và group

```c
–––––––––––––––––––––––––––––––––––––––––––––– users_groups/ugid_functions.c
#include <pwd.h>
#include <grp.h>
#include <ctype.h>
#include "ugid_functions.h" /* Declares functions defined here */
char * /* Return name corresponding to 'uid', or NULL on error */
userNameFromId(uid_t uid)
{
    struct passwd *pwd;
    pwd = getpwuid(uid);
    return (pwd == NULL) ? NULL : pwd->pw_name;
}
uid_t /* Return UID corresponding to 'name', or -1 on error */
userIdFromName(const char *name)
{
    struct passwd *pwd;
    uid_t u;
    char *endptr;

    if (name == NULL || *name == '\0') /* On NULL or empty string */
        return -1;                     /* return an error */
    u = strtol(name, &endptr, 10);     /* As a convenience to caller */
    if (*endptr == '\0')               /* allow a numeric string */
        return u;
    pwd = getpwnam(name);
    if (pwd == NULL)
        return -1;
    return pwd->pw_uid;
}
char * /* Return name corresponding to 'gid', or NULL on error */
groupNameFromId(gid_t gid)
{
    struct group *grp;
    grp = getgrgid(gid);
    return (grp == NULL) ? NULL : grp->gr_name;
}
gid_t /* Return GID corresponding to 'name', or -1 on error */
groupIdFromName(const char *name)
{
    struct group *grp;
    gid_t g;
    char *endptr;
    if (name == NULL || *name == '\0') /* On NULL or empty string */
        return -1;                     /* return an error */
    g = strtol(name, &endptr, 10);     /* As a convenience to caller */
    if (*endptr == '\0')               /* allow a numeric string */
        return g;
    grp = getgrnam(name);
    if (grp == NULL)
        return -1;
    return grp->gr_gid;
}
–––––––––––––––––––––––––––––––––––––––––––––– users_groups/ugid_functions.c
```

### Duyệt toàn bộ bản ghi trong file password và group file

Các hàm `setpwent()`, `getpwent()`, và `endpwent()` được dùng để duyệt tuần tự các bản ghi trong file password.

```c
#include <pwd.h>
struct passwd *getpwent(void);
                   Returns pointer on success, or NULL on end of stream or error
void setpwent(void);
void endpwent(void);
```

Hàm `getpwent()` trả về từng bản ghi trong file password một lần một, trả về `NULL` khi không còn bản ghi nào (hoặc xảy ra lỗi). Khi gọi lần đầu, `getpwent()` tự động mở file password. Khi làm xong, ta gọi `endpwent()` để đóng file.

Ta có thể duyệt toàn bộ file password in tên đăng nhập và user ID bằng code sau:

```c
struct passwd *pwd;
while ((pwd = getpwent()) != NULL)
    printf("%-8s %5ld\n", pwd->pw_name, (long) pwd->pw_uid);
endpwent();
```

Việc gọi `endpwent()` là cần thiết để các lần gọi `getpwent()` tiếp theo (có thể ở phần khác trong chương trình hoặc trong hàm thư viện ta gọi) sẽ mở lại file password và bắt đầu từ đầu. Mặt khác, nếu đang ở giữa chừng file, ta có thể dùng `setpwent()` để bắt đầu lại từ đầu.

Các hàm `getgrent()`, `setgrent()`, và `endgrent()` thực hiện các tác vụ tương tự cho group file. Chúng ta bỏ qua các prototype của các hàm này vì chúng tương tự các hàm file password mô tả trên; xem manual page để biết chi tiết.

### Lấy bản ghi từ shadow password file

Các hàm sau dùng để lấy từng bản ghi từ shadow password file và để duyệt toàn bộ bản ghi trong đó.

```c
#include <shadow.h>
struct spwd *getspnam(const char *name);
                       Returns pointer on success, or NULL on not found or error
struct spwd *getspent(void);
                   Returns pointer on success, or NULL on end of stream or error
void setspent(void);
void endspent(void);
```

Chúng ta sẽ không mô tả chi tiết các hàm này vì cách hoạt động tương tự các hàm file password tương ứng. (Các hàm này không được quy định trong SUSv3 và không có trên tất cả các triển khai UNIX.)

Các hàm `getspnam()` và `getspent()` trả về pointer đến structure kiểu `spwd`. Structure này có dạng như sau:

```c
struct spwd {
    char *sp_namp;          /* Login name (username) */
    char *sp_pwdp;          /* Encrypted password */
    /* Các trường còn lại hỗ trợ "password aging", tính năng tùy chọn
       buộc người dùng thay đổi mật khẩu định kỳ, để ngay cả khi kẻ tấn
       công lấy được mật khẩu, nó cũng sẽ hết hạn sử dụng. */
    long sp_lstchg;         /* Time of last password change
                               (days since 1 Jan 1970) */
    long sp_min;            /* Min. number of days between password changes */
    long sp_max;            /* Max. number of days before change required */
    long sp_warn;           /* Number of days beforehand that user is
                               warned of upcoming password expiration */
    long sp_inact;          /* Number of days after expiration that account
                               is considered inactive and locked */
    long sp_expire;         /* Date when account expires
                               (days since 1 Jan 1970) */
    unsigned long sp_flag;  /* Reserved for future use */
};
```

Chúng ta minh họa cách dùng `getspnam()` trong Listing 8-2.

---

## 8.5 Mã Hóa Mật Khẩu và Xác Thực Người Dùng

Một số ứng dụng yêu cầu người dùng tự xác thực. Xác thực thường có dạng tên người dùng (login name) và mật khẩu. Ứng dụng có thể duy trì cơ sở dữ liệu tên người dùng và mật khẩu riêng. Tuy nhiên, đôi khi cần hoặc tiện hơn khi cho phép người dùng nhập tên đăng nhập và mật khẩu chuẩn đã định nghĩa trong `/etc/passwd` và `/etc/shadow`. (Đối với phần còn lại của mục này, chúng ta giả định hệ thống có shadow password được bật, do đó mật khẩu mã hóa lưu trong `/etc/shadow`.) Các ứng dụng mạng cung cấp đăng nhập từ xa như `ssh` và `ftp` là những ví dụ điển hình. Các ứng dụng này phải xác thực tên người dùng và mật khẩu theo cách tương tự chương trình `login` chuẩn.

Vì lý do bảo mật, các hệ thống UNIX mã hóa mật khẩu bằng thuật toán mã hóa một chiều (one-way), nghĩa là không có cách nào tái tạo lại mật khẩu gốc từ dạng đã mã hóa. Do đó, cách duy nhất để xác thực mật khẩu ứng cử là mã hóa nó theo cùng phương pháp và xem kết quả mã hóa có khớp với giá trị lưu trong `/etc/shadow` không. Thuật toán mã hóa được đóng gói trong hàm `crypt()`.

```c
#define _XOPEN_SOURCE
#include <unistd.h>
char *crypt(const char *key, const char *salt);
                         Returns pointer to statically allocated string containing
                                 encrypted password on success, or NULL on error
```

Thuật toán `crypt()` nhận một `key` (tức là mật khẩu) tối đa 8 ký tự, và áp dụng một biến thể của thuật toán Data Encryption Standard (DES) lên đó. Đối số `salt` là chuỗi 2 ký tự có giá trị dùng để làm xáo trộn (vary) thuật toán — một kỹ thuật được thiết kế để làm cho việc crack mật khẩu mã hóa khó hơn. Hàm trả về pointer đến chuỗi 13 ký tự được cấp phát tĩnh chứa mật khẩu đã mã hóa.

> Chi tiết về DES có tại http://www.itl.nist.gov/fipspubs/fip46-2.htm. Như đã nói trước đó, các thuật toán khác có thể được dùng thay vì DES. Ví dụ, MD5 tạo ra chuỗi 34 ký tự bắt đầu bằng dấu đô la (`$`), cho phép `crypt()` phân biệt mật khẩu mã hóa DES với mật khẩu mã hóa MD5.

> Trong thảo luận về mã hóa mật khẩu, chúng ta dùng từ "mã hóa" một cách hơi lỏng lẻo. Chính xác hơn, DES dùng chuỗi mật khẩu cho trước làm encryption key để mã hóa một chuỗi bit cố định, trong khi MD5 là loại hàm hashing phức tạp. Kết quả trong cả hai trường hợp đều giống nhau: một phép biến đổi không thể đọc và không thể đảo ngược của mật khẩu đầu vào.

Cả đối số `salt` lẫn mật khẩu mã hóa đều gồm các ký tự được chọn từ tập 64 ký tự `[a-zA-Z0-9/.]`. Do đó, đối số `salt` 2 ký tự có thể làm cho thuật toán mã hóa biến thể theo bất kỳ cách nào trong số 64 × 64 = 4096 cách khác nhau. Điều này có nghĩa là thay vì mã hóa trước toàn bộ từ điển và kiểm tra mật khẩu mã hóa với tất cả các từ, kẻ tấn công sẽ cần kiểm tra mật khẩu với 4096 phiên bản mã hóa của từ điển.

Mật khẩu mã hóa do `crypt()` trả về chứa một bản sao của giá trị `salt` gốc trong hai ký tự đầu. Điều này có nghĩa là khi mã hóa mật khẩu ứng cử, ta có thể lấy giá trị `salt` thích hợp từ giá trị mật khẩu mã hóa đã lưu trong `/etc/shadow`. (Các chương trình như `passwd(1)` tạo giá trị `salt` ngẫu nhiên khi mã hóa mật khẩu mới.) Thực tế, hàm `crypt()` bỏ qua mọi ký tự trong chuỗi `salt` sau hai ký tự đầu. Do đó, ta có thể truyền chính mật khẩu mã hóa làm đối số `salt`.

Để dùng `crypt()` trên Linux, ta phải biên dịch chương trình với tùy chọn `-lcrypt`, để chúng được link với thư viện `crypt`.

### Chương trình ví dụ

Listing 8-2 minh họa cách dùng `crypt()` để xác thực người dùng. Chương trình này đầu tiên đọc tên người dùng rồi lấy bản ghi password tương ứng và (nếu có) bản ghi shadow password. Chương trình in thông báo lỗi và thoát nếu không tìm thấy bản ghi password, hoặc nếu chương trình không có quyền đọc shadow password file (yêu cầu đặc quyền superuser hoặc là thành viên của group `shadow`). Sau đó chương trình đọc mật khẩu của người dùng bằng hàm `getpass()`.

```c
#define _BSD_SOURCE
#include <unistd.h>
char *getpass(const char *prompt);
                     Returns pointer to statically allocated input password string
                                                      on success, or NULL on error
```

Hàm `getpass()` đầu tiên tắt tính năng echo và mọi xử lý ký tự terminal đặc biệt (như ký tự interrupt, thường là Control-C). (Chúng ta giải thích cách thay đổi các terminal setting này trong Chương 62.) Sau đó nó in chuỗi được trỏ bởi `prompt`, đọc một dòng đầu vào và trả về chuỗi đầu vào kết thúc bằng null với ký tự newline cuối đã bỏ, như kết quả hàm. (Chuỗi này được cấp phát tĩnh nên sẽ bị ghi đè khi gọi `getpass()` tiếp theo.) Trước khi trả về, `getpass()` khôi phục terminal setting về trạng thái ban đầu.

Sau khi đọc mật khẩu bằng `getpass()`, chương trình trong Listing 8-2 xác thực mật khẩu đó bằng cách dùng `crypt()` để mã hóa nó và kiểm tra xem chuỗi kết quả có khớp với mật khẩu mã hóa được ghi trong shadow password file không. Nếu mật khẩu khớp, ID của người dùng được hiển thị:

```
$ su    Cần đặc quyền để đọc shadow password file
Password:
# ./check_password
Username: mtk
Password:   Ta gõ mật khẩu, không hiển thị trên màn hình
Successfully authenticated: UID=1000
```

Chương trình trong Listing 8-2 kích thước mảng ký tự chứa tên người dùng bằng giá trị trả về của `sysconf(_SC_LOGIN_NAME_MAX)`, trả về kích thước tối đa của username trên hệ thống. Chúng ta giải thích cách dùng `sysconf()` trong Mục 11.2.

**Listing 8-2:** Xác thực người dùng với shadow password file

```c
–––––––––––––––––––––––––––––––––––––––––––––– users_groups/check_password.c
#define _BSD_SOURCE  /* Get getpass() declaration from <unistd.h> */
#define _XOPEN_SOURCE /* Get crypt() declaration from <unistd.h> */
#include <unistd.h>
#include <limits.h>
#include <pwd.h>
#include <shadow.h>
#include "tlpi_hdr.h"
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
    if (lnmax == -1)          /* If limit is indeterminate */
        lnmax = 256;          /* make a guess */
    username = malloc(lnmax);
    if (username == NULL)
        errExit("malloc");
    printf("Username: ");
    fflush(stdout);
    if (fgets(username, lnmax, stdin) == NULL)
        exit(EXIT_FAILURE);   /* Exit on EOF */
    len = strlen(username);
    if (username[len - 1] == '\n')
        username[len - 1] = '\0'; /* Remove trailing '\n' */
    pwd = getpwnam(username);
    if (pwd == NULL)
        fatal("couldn't get password record");
    spwd = getspnam(username);
    if (spwd == NULL && errno == EACCES)
        fatal("no permission to read shadow password file");
    if (spwd != NULL)              /* If there is a shadow password record */
        pwd->pw_passwd = spwd->sp_pwdp; /* Use the shadow password */
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
––––––––––––––––––––––––––––––––––––––––––––––– users_groups/check_password.c
```

Listing 8-2 minh họa một điểm bảo mật quan trọng. Các chương trình đọc mật khẩu nên lập tức mã hóa mật khẩu đó và xóa phiên bản chưa mã hóa khỏi bộ nhớ. Điều này giảm thiểu khả năng chương trình bị crash tạo ra core dump file có thể đọc để khám phá mật khẩu.

> Còn có các cách khác mà mật khẩu chưa mã hóa có thể bị lộ. Ví dụ, mật khẩu có thể được đọc từ swap file bởi chương trình có đặc quyền nếu virtual memory page chứa mật khẩu bị swap out. Ngoài ra, một process có đủ đặc quyền có thể đọc `/dev/mem` (thiết bị ảo trình bày physical memory của máy tính như một dòng byte tuần tự) nhằm tìm mật khẩu.

> Hàm `getpass()` xuất hiện trong SUSv2, đánh dấu là LEGACY, chỉ ra rằng tên hàm gây hiểu nhầm và chức năng mà nó cung cấp thực ra dễ tự triển khai. Đặc tả của `getpass()` đã bị loại bỏ trong SUSv3. Tuy nhiên nó vẫn xuất hiện trên hầu hết các triển khai UNIX.

---

## 8.6 Tóm Tắt

Mỗi người dùng có tên đăng nhập duy nhất và numeric user ID đi kèm. Người dùng có thể thuộc một hoặc nhiều group, mỗi group cũng có tên duy nhất và numeric identifier đi kèm. Mục đích chính của các identifier này là xác lập quyền sở hữu các tài nguyên hệ thống (ví dụ: file) và quyền truy cập chúng.

Tên và ID của người dùng được định nghĩa trong file `/etc/passwd`, cũng chứa các thông tin khác về người dùng. Group membership của người dùng được định nghĩa bởi các trường trong `/etc/passwd` và `/etc/group`. Một file khác, `/etc/shadow`, chỉ có thể đọc bởi các process có đặc quyền, được dùng để tách biệt thông tin mật khẩu nhạy cảm khỏi thông tin người dùng công khai trong `/etc/passwd`. Nhiều hàm thư viện được cung cấp để lấy thông tin từ mỗi file này.

Hàm `crypt()` mã hóa mật khẩu theo cách tương tự chương trình `login` chuẩn, hữu ích cho các chương trình cần xác thực người dùng.

---

## 8.7 Bài Tập

**8-1.** Khi thực thi code sau, ta thấy nó hiển thị cùng một số hai lần, mặc dù hai người dùng có ID khác nhau trong file password. Tại sao?

```c
printf("%ld %ld\n", (long) (getpwnam("avr")->pw_uid),
                    (long) (getpwnam("tsr")->pw_uid));
```

**8-2.** Triển khai `getpwnam()` bằng cách dùng `setpwent()`, `getpwent()`, và `endpwent()`.
