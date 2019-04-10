---
title: Phân vùng bảng, index trong Oracle Database [Phần 1]
tags:
  - oracle
  - table partition
  - index partition
categories:
  - Oracle
toc: true
toc_label: "Table of Contents"
toc_icon: "file-alt"
---

# Giới thiệu về Phân vùng

Phân vùng giải quyết vấn đề quan trọng trong việc hỗ trợ xử lý các bảng và indexes có kích thước rất lớn bằng cách phân chia chúng thành các phần nhỏ hơn để dễ quản lý. Đối với các ứng dụng, các câu truy vấn SQL và câu lệnh DML không cần phải chỉnh sửa để có thể truy xuất phân vùng. Sau khi định nghĩa phân vùng, các câu lệnh DDL có thể thao tác trực tiếp tới từng phân vùng riêng lẻ của bảng và indexes. Phân vùng là cách đơn giản hoá việc quản lý các cơ sở dữ liệu lớn.

Mỗi phân vùng của bảng và indexes phải có các thuộc tính logical giống nhau như tên cột, kiểu dữ liệu và các ràng buộc (constraints), tuy nhiên mỗi phân vùng lại có các thuộc tính physical độc lập với nhau như pctfree, pctused và tablespaces.

Việc chia phân vùng hữu ích đối với nhiều kiểu ứng dụng khác nhau, đặc biệt là các ứng dụng quản lý lượng lớn dữ liệu. Các hệ thống OLTP (on-line transactional processing) thông thường được hưởng lợi từ việc cải thiện khả năng quản lý và tính sẵn sàng, trong khi đó các hệ thống OLAP (on-line analytical processing) hưởng lợi từ việc cải thiện hiệu năng và khả năng quản lý.

__Lợi thế của việc chia phân vùng:__
- Phân vùng cho phép các thao tác quản lý dữ liệu như load dữ liệu, create và rebuild indexes, sao lưu/phục hồi ở mức phân vùng chứ không cần thao tác trên toàn bộ bảng. Điều này giúp giảm đáng kể thời gian cho những hoạt động đó.
- Phân vùng cải thiện hiệu suất truy vấn. Trong nhiều trường hợp, kết quả của truy vấn có thể có được bằng cách truy cập vào một tập hợp con của các phân vùng chứ không phải toàn bộ bảng. Kỹ thuật này còn được gọi là `partition pruning`.
- Phân vùng làm giảm đáng kể thời gian downtime trong các hoạt động bảo trì.
- Phân vùng các bảng và indexes giúp tăng tính sẵn sàng đối với các database quan trọng, giảm thiểu thời gian phục hồi cũng như các ảnh hưởng của lỗi.
- Phân vùng cho phép đồng thời bảo trì trên nhiều phân vùng bảng hoặc indexes khác nhau. Có thể chạy đồng thời các hành động SELECT, DML đối với phân vùng mà không ảnh hưởng đến quá trình bảo trì.

Việc chia phân vùng cho phép truy cập dữ liệu nhanh hơn đối với Oracle database. Cho dù database có 10GB hoặc 10TB dữ liệu, phân vùng có thể giúp tăng khả năng truy xuất tùy theo độ lớn của database. Phân vùng có thể được thực hiện mà không cần bất kỳ thay đổi nào trên lớp ứng dụng. Ví dụ, một bảng có thể chuyển từ nonpartitioned sang partitioned mà không cần chỉnh sửa bất kỳ câu `SELECT` hoặc DML nào để có thể truy cập dữ liệu. 

# Khái quát về Phân vùng 

Phân vùng giúp tăng hiệu suất, khả năng quản lý và tính sẵn sàng của nhiều loại ứng dụng. Phân vùng cho phép các bảng, chỉ mục được chia thành các phần nhỏ hơn giúp việc quản lý và truy cập các database objects đó ở mức độ chi tiết hơn. Oracle cung cấp nhiều chiến lược phân vùng để giải quyết các bài toán khác nhau cho doanh nghiệp. Bời vì nó hoàn toàn trong suốt nên phân vùng có thể được áp dụng cho hầu hết mọi ứng dụng mà không cần bất kỳ thay đổi nào tốn kém thời gian.


## Tổng quan về Phân vùng

### Cơ bản về phân vùng

_Hình 2-1. Phân vùng trong bảng._

![](/img/vldbg008.gif)


### Khoá phân vùng (Partition Key)
Mỗi hàng thuộc bảng đã được phân vùng chỉ thuộc về một phân vùng duy nhất. Khoá phân vùng là tập hợp một hoặc nhiều cột xác định phân vùng nơi từng hàng được lưu trữ. Oracle sẽ trực tiếp xử lý luồng của các câu lệnh INSERT, UPDATE, DELETE vào các phân vùng thích hợp thông qua khoá phân vùng.

**Một khoá phân vùng:**
- Bao gồm danh sách có thứ tự từ 1 đến 16 cột.
- Không thể chứa LEVEL, ROWID, MLSLABEL pseudocolumn hoặc một cột kiểu ROWID.
- Không thể chứa cột NULL.

### Phân vùng bảng
- Tất cả các bảng đều có thể được phân vùng trừ các bảng có chứa các cột có kiểu LONG hoặc RAW LONG. Tuy nhiên có thể sử dụng các bảng có kiểu CLOB, BLOB.

#### Khi nào thực hiện phân vùng bảng?

- Bảng lớn hơn 2 GB luôn được coi là ứng viên cho phân vùng.
- Bảng chứa dữ liệu lịch sử, những dữ liệu mới sẽ được thêm vào phân vùng mới.
- Khi nội dung của bảng cần được phân phối trên nhiều loại thiết bị storage khác nhau.

#### Khi nào thực hiện phân vùng index?

- Tránh việc phải rebuilding toàn bộ index khi dữ liệu bị xóa bỏ.
- Thực hiện bảo trì trên một phần dữ liệu mà không gây ảnh hưởng đến toàn bộ index (invalid).
- Giảm ảnh hưởng của lệch index gây ra trên cột có giá trị tăng đơn điệu.

## Lợi ích của việc phân vùng

Phân vùng có thể mang lại nhiều lợi ích cho ứng dụng bằng cách cải thiện hiệu năng, khả năng quản lý, tính sẵn sàng. 

### Phân vùng giúp tăng hiệu năng

Phân vùng cung cấp nhiều cách để tăng hiệu năng bằng cách giới hạn lượng dữ liệu cần xử lý, thực thi câu lệnh parallel.

#### Partition Pruning

Partition Pruning (hay phân vùng tỉa) là đơn giản nhất và cũng là cách tăng hiệu năng đáng kể nhất khi sử dụng partition. Giả sử một ứng dụng chứa bảng Orders chứa dữ liệu lịch sử đặt hàng, bảng này được chia phân vùng theo tuần. Một câu lệnh truy vấn cần lấy báo cáo các đơn đặt hàng trong một tuần sẽ chỉ cần truy cập tới một phân vùng của của bảng Orders. Nếu bảng dữ liệu này chứa 2 năm dữ liệu lịch sử thì câu lệnh này chỉ cần truy cập một phân vùng thay vì 104 phân vùng. Như vậy câu lệnh truy vấn này có khả năng tăng tốc độ thực thi lên hơn 100 lần.

#### Partition-Wise Joins

Phân vùng cũng có thể cải thiện hiệu suất của phép join nhiều bảng bằng cách sử dụng kỹ thuật được gọi là _partition-wise joins_. Partition-wise joins có thể áp dụng khi 2 bảng được join với nhau và cả 2 bảng được phân vùng trên join key.

### Phân vùng giúp tăng khả năng quản lý

Phân vùng cho phép phân chia bảng và index thành các vùng nhỏ hơn, dễ quản lý hơn cho DBA. Giả sử khi cần thì DBA có thể backup một tháng dữ liệu nào đó thay vì backup toàn bộ bảng. 

### Phân vùng giúp tăng tính sẵn sàng

Nếu một phân vùng của bảng bị lỗi, các phân vùng khác vẫn truy cập hoàn toàn bình thường. Ứng dụng vẫn có thể thực hiện các giao dịch đối với các phân vùng có sẵn, miễn là nó không cần truy cập vào phân vùng đang bị lỗi.

Kịch bản khác là việc phân vùng cho phép người quản trị có thể lưu trữ các phân vùng trên các vùng đĩa khác nhau, có tốc độ và khả năng sẵn sàng khác nhau. Ngoài ra còn có thể sao lưu và phục hồi từng phân vùng riêng biệt, độc lập với các phân vùng khác trong bảng.

## Cách thức phân vùng

Oracle Partitioning cung cấp 3 phương thức chia phân vùng cơ bản để kiểm soát việc phân chia dữ liệu:

- Range
- Hash
- List

Bằng cách sử dụng các phương thức trên, một bảng có thể được chia phân vùng như một phân vùng danh sách đơn (single list) hoặc một phân vùng tổng hợp (composite).

- Phân vùng Single-Level
- Phân vùng Composite

Mỗi cách thức phân vùng có những lợi thế và cân nhắc thiết kế khác nhau. Vì vậy, mỗi cách thức phù hợp hơn cho một tình huống cụ thể.

### Phân vùng Single-Level

Một bảng được định nghĩa bởi một trong các phương thức phân vùng sau, sử dụng một hoặc nhiều cột làm khóa phân vùng:

- Phân vùng theo phạm vi (Range Partitioning)
- Phân vùng theo danh sách (List Partitioning)
- Phân vùng băm (Hash Partitioning)

_Hình 2-2 Phân vùng List, Range và Hash._

![](/img/vldbg005.gif)

#### Phân vùng phạm vi (Range Partition)

Phân vùng phạm vi dữ liệu dựa trên phạm vi của khoá phân vùng mà ta thiết lập. Đây là loại phân vùng phổ biến nhất và thường được sử dụng với kiểu dữ liệu `date`. Ví dụ, đối với một bảng có cột `date` làm khóa phân vùng, phân vùng `January-2010` có thể chứa các hàng với khóa phân vùng có giá trị từ `01-Jan-2010` đến `31-Jan-2010`.

Khi sử dụng phân vùng phạm vi:

- Mỗi phân vùng đều có một mệnh đề `VALUES LESS THAN` chỉ định giới hạn trên của phân vùng. Bất kỳ giá trị của khóa phân vùng nào lớn hơn hoặc bằng giá trị này đều được thêm vào phân vùng tiếp theo cao hơn.
- Tất cả các phân vùng, trừ phân vùng đầu tiên đều có một giá trị giới hạn dưới ngầm định bởi mệnh đề `VALUES LESS THAN` của phân vùng trước đó.
- Một giá trị `MAXVALUES` (lớn nhất theo nghĩa đen) có thể được định nghĩa cho phân vùng cao nhất. `MAXVALUE` đại diện cho một giá trị vô hạn, cao hơn bất kỳ khóa phân vùng có thể có, bao gồm cả giá trị `NULL`.

_Ví dụ tạo một bảng `sale_range` được phân vùng phạm vi theo trường `sale_date`:_

```sql
	CREATE TABLE sales_range 
	(salesman_id NUMBER(5), 
	salesman_name VARCHAR2(30), 
	sales_amount NUMBER(10), 
	sales_date DATE)
	PARTITION BY RANGE(sales_date) 
	(
	PARTITION sales_jan2019 VALUES LESS THAN(TO_DATE('02/01/2019','DD/MM/YYYY')),
	PARTITION sales_feb2019 VALUES LESS THAN(TO_DATE('03/01/2019','DD/MM/YYYY')),
	PARTITION sales_mar2019 VALUES LESS THAN(TO_DATE('04/01/2019','DD/MM/YYYY')),
	PARTITION sales_apr2019 VALUES LESS THAN(TO_DATE('05/01/2019','DD/MM/YYYY'))
	);
```

#### Phân vùng danh sách (List Partition)

Phân vùng theo danh sách cho phép chỉ rõ hàng nào thuộc phân vùng nào. Khác với phân vùng theo phạm vi, phân vùng theo danh sách chỉ rõ giá trị cho khoá phân vùng.

Ưu điểm của phân vùng danh sách là có thể nhóm và tổ chức các dữ liệu chưa được sắp xếp hoặc dữ liệu không liên quan một cách tự nhiên. 

Ví dụ, một bảng chứa cột `region` là khóa phân vùng, phân vùng `sales_east` có thể chứa giá trị của `New York`, `Virginia` và `Florida`.

_Ví dụ sau, bảng sales_list được phân vùng bảng bán hàng theo khu vực:_

```sql
	CREATE TABLE sales_list
	(salesman_id NUMBER(5), 
	salesman_name VARCHAR2(30),
	sales_state VARCHAR2(20),
	sales_amount NUMBER(10), 
	sales_date DATE)
	PARTITION BY LIST(sales_state)
	(
	PARTITION sales_west VALUES('California', 'Hawaii'),
	PARTITION sales_east VALUES ('New York', 'Virginia', 'Florida'),
	PARTITION sales_central VALUES('Texas', 'Illinois')
	PARTITION sales_other VALUES(DEFAULT)
	);
```

Mặc định, tất cả các hàng dữ liệu không được map với giá trị khóa phân vùng nào được khai báo trước sẽ được đặt vào phân vùng mặc định là `DEFAULT`.

#### Phân vùng băm (Hash Parition)

Phân vùng băm dễ dàng cho phép phân vùng dữ liệu mà không phù hợp với phân vùng phạm vi và phân vùng danh sách. Phân vùng băm dễ dàng thực hiện với cú pháp đơn giản. Phân vùng băm tốt hơn sơ với phân vùng phạm vi khi:

- Không biết trước có bao nhiêu dữ liệu trong một phạm vi.
- Các phạm vi của phân vùng phạm vi khác nhau hoặc khó cân bằng.
- Phân vùng phạm vi có thể nhóm những dữ liệu không mong muốn.
- Các tính năng hiệu năng như DML song song, phân vùng tỉa (partition pruning), gộp phân vùng (partition-wise joins) là quan trọng.

Các khái nhiệm `splitting`, `dropping`, `merging` không áp dụng cho phân vùng băm. Thay vào đó phân vùng băm có thể được thêm vào và kết hợp lại (added and coalesced).

_Ví dụ về phân vùng băm:_

```sql
	CREATE TABLE sales_hash
	(salesman_id NUMBER(5),
	salesman_name VARCHAR2(30),
	sales_amount NUMBER(10),
	week_no NUMBER(2))
	PARTITION BY HASH(salesman_id)
	PARTITIONS 4
	STORE IN (data1, data2, data3, data4);
```

### Phân vùng tổng hợp (Composite Partition)

Phân vùng tổng hợp phân chia dữ liệu bằng cách sử dụng phương pháp phân vùng phạm vi, trong mỗi phân vùng, phân vùng phụ (subpartition) sử dụng phương pháp phân vùng danh sách hoặc phân vùng băm. Phân vùng tổng hợp phạm vi-băm (`composite range-hash`) cải tiến khả năng quản lý của phân vùng phạm vi và khả năng phân chia, định vị dữ liệu của phân vùng băm. Phân vùng tổng hợp phạm vi-danh sách (`composite range-list`) cải tiến khả năng quản lý của phân vùng phạm vi và cơ chế điều khiển rõ ràng của phân vùng danh sách đối với phân vùng phụ.

Phân vùng tổng hợp hỗ trợ các thao tác trên dữ liệu lịch sử như thêm phân vùng phạm vi mới, ngoài ra cung cấp tốt hơn khả năng DML song song và độ chi tiết tốt hơn thông qua phân vùng phụ.

_Ví dụ tạo phân vùng tổng hợp phạm vi-băm (`Range-Hash`):_

```sql
	CREATE TABLE sales_composite 
	(salesman_id NUMBER(5), 
	salesman_name VARCHAR2(30), 
	sales_amount NUMBER(10), 
	sales_date DATE)
	PARTITION BY RANGE(sales_date) 
	SUBPARTITION BY HASH(salesman_id)
	SUBPARTITION TEMPLATE(
	SUBPARTITION sp1 TABLESPACE ts1,
	SUBPARTITION sp2 TABLESPACE ts2,
	SUBPARTITION sp3 TABLESPACE ts3,
	SUBPARTITION sp4 TABLESPACE ts4)
	(PARTITION sales_jan2000 VALUES LESS THAN(TO_DATE('02/01/2000','DD/MM/YYYY'))
	PARTITION sales_feb2000 VALUES LESS THAN(TO_DATE('03/01/2000','DD/MM/YYYY'))
	PARTITION sales_mar2000 VALUES LESS THAN(TO_DATE('04/01/2000','DD/MM/YYYY'))
	PARTITION sales_apr2000 VALUES LESS THAN(TO_DATE('05/01/2000','DD/MM/YYYY'))
	PARTITION sales_may2000 VALUES LESS THAN(TO_DATE('06/01/2000','DD/MM/YYYY')));
```

Câu lệnh trên tạo bảng `sales_composite` chỉ ra phân vùng phạm vi với khóa phân vùng là `sales_date` và phân vùng băm phụ với khóa phân vùng là `sale_id`. Khi sử dụng template này, Oracle đặt tên phân vùng phụ bằng cách ghép `<tên phân vùng>_<tên phân vùng phụ>` từ template ta định nghĩa. Oralce đặt phân vùng phụ trong tablespace đã được định nghĩa. Trong câu lệnh trên, phân vùng `sales_jan2000_sp1` được tạo trong tablespace ts1, phân vùng `sales_jan2000_sp4` được tạo trong tablespace ts4. Hình bên dưới mô tả cho ví dụ:

_Hình 2_3 Phân vùng phạm vi-danh sách `Range-Hash`._ 

Phân vùng tổng hợp có một số loại như sau:

- Composite Range-Range Partitioning
- Composite Range-Hash Partitioning
- Composite Range-List Partitioning
- Composite List-Range Partitioning
- Composite List-Hash Partitioning
- Composite List-List Partitioning

![](/img/cncpt157.gif)

Kết thúc phần 1, phần sau mình sẽ viết về phân vùng đối với Indexes.
