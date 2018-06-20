# Thực hiện việc ghi đa luồng trong SQLite gần như cùng một thời điểm 

Các bạn thân mến,

Như bạn đã biết, SQlite là cơ sở dữ liệu đơn luồng, chúng được nhúng mặc định trong hệ điều hành Linux. Có rất nhiều nghiên cứ về việc sử dụng SQLite để lưu trữ dữ liệu. Và cũng đã có nhiều nghiên cứ về cách làm thế nào truy cập cơ sở dữ liệu SQLite để phục vụ cho việc ghi đa luồng. Tôi sẽ chia sẻ nghiên cứu nhỏ của mình về việc thực hiện ghi nhiều luồng trên cơ sở dữ liệu SQLite.

Hãy cùng xem các ưu và nhược điểm của SQLite.

### Ưu điểm.

- SQLite được viết bằng ngôn ngữ lập trình C thuần. Vì vậy, việc truy cập vào ổ đĩa hoặc bộ nhớ cơ sở dữ liệu và xử lý dữ liệu là nhanh nhất. Bạn có thể liên tưởng điều này giống như việc sử dụng ổ SSD vậy. 

- SQLite hỗ trợ bộ nhớ trong. Bộ nhớ trong của SQLite nhanh gấp gần 2 lần nếu như bạn hiểu về vấn đề phân trang. Nó đủ nhanh.

- SQLite là đơn luồng. Vì vậy, nguy cơ hỏng dữ liệu được gỉam thiểu tối đa.

- Cơ sở dữ liệu SQLite nằm trong một file. Vì vậy, chúng ta có thể di chuyển cơ sở dữ liệu và truy cập bởi nền tảng khác rất dễ dàng. 

- SQLite không yêu cầu thành phần quản trị đối với người dùng cuối. 

- Đa nền tảng. SQLite có thể được sử dụng trên hầu hết tất cả các nền tảng hệ điều hành. 

- Mã nguồn mở  everywhere!

- và vân vân ...

### Nhược điểm.

- Như chúng ta đã đề cập, SQLite là đơn luồng. Vì vậy, tại mỗi thời điểm, SQLite chỉ có thể thực hiện một thao tác ghi.

- Như chúng ta đã đề cập, SQLite lưu trữ cơ sở dữ liệu trong một file. Do vậy, toàn bộ cơ sở dữ liệu được khóa lại trong quá trình ghi. Điều này rất không phù hợp đối với cơ sở dữ liệu có lượng truy cập lớn và thường xuyên.

- Không yêu cầu xác thực mức ứng dụng.

### Thực hiện ghi đa luồng cùng một lúc.

Bây giờ ta sẽ cố gắng tìm ra mẹo nhỏ để làm cho các thao tác ghi được thực hiện gần như cùng một lúc.

**Chú ý: SQLite không bao giờ cho phép bạn thực hiện ROW LEVEL LOCK. Do vậy, ta không cần phải lãng phí thời gian để tìm kiếm về nó.**

Tất nhiên nó có thể tác động đến hiệu năng nếu bạn thực hiện hành động ghi trên phần lớn bảng. Nhưng luồng thứ 2 sẽ phải không chờ lâu cho đến khi hành động đầu tiên kết thúc.

![](https://media.licdn.com/dms/image/C5612AQFU1Qb1s5ET0w/article-inline_image-shrink_1000_1488/0?e=2128896000&v=beta&t=l4oDD044Ifz5bnf_9Z0mLqE8H10w5nIFL0AOojqQAw0)

Bời vì như bạn biết index B TREE là rất nhanh. Và hành động GHI của bạn sẽ thực hiện dựa theo ID từng dòng mà đã được index theo mặc định.

Ví dụ:

**delete from table where name='Fariz'** // sẽ khoá toàn bộ dữ liệu chính trong 10 giây.

THAY BẰNG:

**insert into tepm.Lock select rowid,'processid' from table where name = 'Fariz';** // nó sẽ chỉ khóa cơ sở dữ liệu tạm thời trong 10 giây.

**delete from table where ROWID in ( select rowid from tepm.Lock where name='fariz' and processid='XYZ' )** // việc xóa sẽ khóa file cơ sở dữ liệu trong 0,001 giây.

Ta đã hoàn thành thử nghiệm nhỏ và kết quả là rất tốt. Ta cũng có thể thể hoàn thành 2 hành động cập nhật tương tự trong 11 giây, mỗi giao dịch mất 10 giây cho bảng bao gồm 40 triệu dòng.
