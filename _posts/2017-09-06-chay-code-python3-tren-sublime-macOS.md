---
title: Chạy code Python3 trực tiếp trên Sublime3 macOS
tags: [til, macos]
classes: wide
categories: [Today I Learned, macOS]
---

Đối với macOS, mặc định Sublime cho phép mình build code trên python2 khi ta chỉ cần bấm `cmd (⌘) + b`. Khi cần build trên python3, mình cần cấu hình thêm một chút.

- Mở **Sublime**
- Chọn menu **Tools** -> **Build-System** -> **New Build System**
- Xác định vị trí execute python3 bằng các sử dụng Terminal và gõ lệnh `which python3`. Kết quả ta được đường dẫn như bên dưới.
```sh
$ which python3
/Library/Frameworks/Python.framework/Versions/3.6/bin/python3
```
- Tạo một file mới trong Sublime và dán vào nội dung sau. Chú ý thay thế đúng giá trị đường dẫn python3.
```sh
{
"cmd": ["/Library/Frameworks/Python.framework/Versions/3.6/bin/python3", "-u", "$file"],
"file_regex": "^[ ]*File \"(...*?)\", line ([0-9]*)",
"selector": "source.python"
}
```
- Lưu file (`cmd (⌘) + s` trên bàn phím) và đặt tên `Python-3.6.sublime-build`.

Bây giờ chuyển sang chế độ build mới, và bạn có thể sử dụng `cmd (⌘) + b` để thực thi code của mình roài.

Dễ ha :D
