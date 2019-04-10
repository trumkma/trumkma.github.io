---
title: Xuất kết quả truy vấn định dạng html bằng SQLPlus
tags: [oracle, sqlplus, til]
classes: wide
categories: [Today I Learned, Linux]
---

Khi không có các công cụ hỗ trợ như Toad, SQL Navigator,... thì việc truy vấn nhiều thông tin trên các bảng qua SQLPlus quả thực là hết sức vất vả.

Mình thì thường dùng tính năng xuất ra file định dạng html sau đó dùng trình duyệt để đọc, thử nhé! 

```sql
set ver off
set term off
set page 0
set markup html on spool on
spool db_info.html
select * from v$database;
spool off
set markup html off spool off
```

Hay đúng không nào :D
