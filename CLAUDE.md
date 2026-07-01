# CLAUDE.md — Hướng dẫn dự án Linux Programming Interface

## Mô tả dự án

Dịch toàn bộ 64 chapter của cuốn sách *The Linux Programming Interface* (Michael Kerrisk) từ tiếng Anh sang tiếng Việt, sau đó phân loại vào các thư mục chủ đề.

---

## Cấu trúc thư mục

```
/home/huyhoang/Desktop/Linux_Programming_Interface/
├── chapters/                          ← Source tiếng Anh gốc (KHÔNG sửa)
│   ├── ch01_history_and_standards.md
│   ├── ... (64 file)
│   └── INDEX.md                       ← Bảng mục lục và phân nhóm chủ đề
├── file_io_and_file_systems/          ← Bản dịch: Ch 4-5, 13-19, 55
├── processes_and_program_execution/   ← Bản dịch: Ch 6, 9, 24-28, 34-37
├── threads_pthreads/                  ← Bản dịch: Ch 29-33
├── signals/                           ← Bản dịch: Ch 20-22
├── memory/                            ← Bản dịch: Ch 7, 49-50
├── interprocess_communication_ipc/    ← Bản dịch: Ch 43-48, 51-54
├── sockets_and_networking/            ← Bản dịch: Ch 56-61
├── timers_and_time/                   ← Bản dịch: Ch 10, 23
├── security_and_privileges/           ← Bản dịch: Ch 38-39
├── shared_libraries/                  ← Bản dịch: Ch 41-42
├── terminals_and_io_models/           ← Bản dịch: Ch 62-64
└── system_info_and_misc/              ← Bản dịch: Ch 1-3, 8, 11-12, 40
```

---

## Quy tắc dịch

1. **Văn xuôi, giải thích** → dịch sang tiếng Việt tự nhiên, dễ hiểu
2. **Tên hàm, system call, macro, flag, struct, biến** → giữ nguyên tiếng Anh, bọc trong backtick (`` `open()` ``, `` `errno` ``)
3. **Thuật ngữ kỹ thuật chuyên ngành** → GIỮ NGUYÊN tiếng Anh:
   - kernel, process, thread, file descriptor, socket, pipe, signal
   - buffer, cache, stack, heap, pointer, callback, handler
   - system call, library function, API, IPC, mutex, semaphore
   - user space, kernel space, virtual memory, page, block
4. **Tiêu đề chương, mục** → dịch tiếng Việt
5. **Code block** → GIỮ NGUYÊN hoàn toàn, không dịch
6. **Tên file** → GIỮ NGUYÊN tiếng Anh (ví dụ: `ch01_history_and_standards.md`)

---

## Ánh xạ chapter → thư mục

Dựa trên phân nhóm trong `chapters/INDEX.md`:

| Thư mục | Chapters |
|---------|---------|
| `system_info_and_misc/` | ch01, ch02, ch03, ch08, ch11, ch12, ch40 |
| `file_io_and_file_systems/` | ch04, ch05, ch13, ch14, ch15, ch16, ch17, ch18, ch19, ch55 |
| `processes_and_program_execution/` | ch06, ch09, ch24, ch25, ch26, ch27, ch28, ch34, ch35, ch36, ch37 |
| `memory/` | ch07, ch49, ch50 |
| `signals/` | ch20, ch21, ch22 |
| `timers_and_time/` | ch10, ch23 |
| `shared_libraries/` | ch41, ch42 |
| `interprocess_communication_ipc/` | ch43, ch44, ch45, ch46, ch47, ch48, ch51, ch52, ch53, ch54 |
| `sockets_and_networking/` | ch56, ch57, ch58, ch59, ch60, ch61 |
| `security_and_privileges/` | ch38, ch39 |
| `threads_pthreads/` | ch29, ch30, ch31, ch32, ch33 |
| `terminals_and_io_models/` | ch62, ch63, ch64 |

---

## Trạng thái tiến độ

### Hoàn thành (57/64 tính đến 2026-06-04)

**file_io_and_file_systems/** (10/10): ch04, ch05, ch13, ch14, ch15, ch16, ch17, ch18, ch19, ch55
**system_info_and_misc/** (7/7): ch01, ch02, ch03, ch08, ch11, ch12, ch40
**memory/** (3/3): ch07, ch49, ch50
**timers_and_time/** (2/2): ch10, ch23
**signals/** (3/3): ch20, ch21, ch22
**processes_and_program_execution/** (11/11): ch06, ch09, ch24, ch25, ch26, ch27, ch28, ch34, ch35, ch36, ch37
**threads_pthreads/** (5/5): ch29, ch30, ch31, ch32, ch33
**security_and_privileges/** (2/2): ch38, ch39
**shared_libraries/** (1/2): ch41 ✓; ch42 đang dịch
**interprocess_communication_ipc/** (6/10): ch43, ch44, ch45, ch48, ch51, ch52 ✓; ch46, ch47, ch53, ch54 đang dịch
**sockets_and_networking/** (4/6): ch56, ch57, ch58, ch59 ✓; ch60, ch61 đang dịch
**terminals_and_io_models/** (0/3): ch62, ch63, ch64 đang dịch

**shared_libraries/** (2/2): ch41, ch42 ✓
**interprocess_communication_ipc/** (10/10): ch43, ch44, ch45, ch46, ch47, ch48, ch51, ch52, ch53, ch54 ✓
**sockets_and_networking/** (6/6): ch56, ch57, ch58, ch59, ch60, ch61 ✓
**terminals_and_io_models/** (3/3): ch62, ch63, ch64 ✓

### HOÀN THÀNH: 64/64 chapter (100%)
