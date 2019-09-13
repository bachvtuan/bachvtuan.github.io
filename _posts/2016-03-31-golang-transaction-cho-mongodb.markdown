---
layout: post
title:  "Golang: Transaction cho Mongodb ?"
date:   2016-03-31 18:34:10 +0700
categories: [golang, mongodb]
---
Mỗi khi tôi triển khai dự án mới thì Mongodb là lựa chọn đầu tiên, Nó khá linh hoạt trong cách lưu trữ và query dữ liệu so với cơ sở dữ liệu Sql. Nhưng vì hiệu năng nó phải hy sinh 1 thứ mà Sql có đó là transaction. Vì thế, một số hệ thông vẫn dùng Sql để đảm bảo tính toàn vẹn dữ liệu . Trong bài này tôi sẽ giải quyết vấn đề này bằng cách sử dụng Golang can thiệp ở phần lập trình nhằm giả lập transaction cho Mongodb.Bản thân Mongodb không hỗ trợ Transaction vì thế  tôi gọi đây là cách giả transaction ở tầng ngôn ngữ lập trình. Vây làm thế nào để giả được? Trước hết ta phải hiểu transaction là gì rồi mới bắt chước được. Mình sẽ không nói rõ transaction là gì, mọi người có thể tìm hiểu bằng Google.

[source link](https://github.com/bachvtuan/Golang-Mongodb-Transaction-Example)


**Ví dụ:** tôi có 1 đoạn code rút tiền được viết bằng thuật toán sau:
```
B1. Yêu cầu xút X tiền.
B2. Lấy số dư hiện tại T, kiểm tra nếu nhỏ hơn X tiền thì thông báo thất bại  rồi thoát nếu không thì qua B3.
B3. Ghi log cho sự kiện này rồi cập nhật lại số dư còn lại với giá tri = T -X.
```
Code này đúng, nếu người này rút tiền lần lượt. Tuy nhiên, nếu cùng 1 lúc anh này có cách nào đó rút cùng 1 lúc( Ví dụ có 2 thẻ rút tiền sử dụng chung 1 tài khoản ) thì lỗ hổng có thể xảy ra.

Đây là trường hợp lỗi : Anh A có 100USD. anh này muốn rút hết 100USD, nếu rút 1 lần thì còn lại 0USD và không thể rút được nữa. Tuy nhiên nếu rút 100USD sử dụng 2 thẻ cùng 1 lúc thì có thể rút thành công 2 lần với tổng số tiền là 200USD.

Vì thế transaction sinh ra để chặn nhiều tiến trình cùng truy cập đoạn code trên. Có thể sửa lại thành
```
db.BeginTransaction()
B1. Yêu cầu xút X tiền.
B2. Lấy số dư hiện tại T, kiểm tra nếu nhỏ hơn X tiền thì thông báo thất bại  rồi thoát nếu không thì qua B3.
B3. Ghi log cho sự kiện này rồi cập nhật lại số dư còn lại với giá tri = T -X.
db.Commit()
```
Bây giờ bạn đã hiểu một chút transaction là gì. Tôi nghĩ bài toán của mình là làm thế nào để quản lý 1 đoạn code mà mỗi lúc chỉ có tiến trình truy cập và xử lý nó, khi nó xử lý xong thì các tiến trình khác mới được xử lý tiếp. Điều này dường như là điều bất khả đối với các ngôn ngữ lập trình mà tôi từng dùng nhưng với Golang, cách giải quyết thật đơn giản.

Giờ hãy tìm hiểu code rút tiền không hỗ trợ transaction
```go
package main

import (
  "io"
  "net/http"
  "fmt"
  "gopkg.in/mgo.v2"
  "gopkg.in/mgo.v2/bson"
)

var global_db *mgo.Database

type Currency struct {
  Id   bson.ObjectId `json:"id" bson:"_id,omitempty"`
  Amount float64 `bson:"amount"`
  Account   string `bson:"account"`
  Code   string `bson:"code"`
}

var count_withdraw = 0

func withdraw(w http.ResponseWriter, r *http.Request) {
  entry := Currency{}
  err := global_db.C("bank").Find(bson.M{"account":  "tuanbach"}).One(&entry)
  

  if err != nil {
    panic(err)
  }

  fmt.Printf("%+v\n", entry)
  //step 2: check if balance is valid to widthdraw
  if entry.Amount < 50.00 {
    fmt.Printf("out_of_balance %d\n", count_withdraw)
    io.WriteString(w, "out_of_balance")
    return
  }

  //step 3: subtract current balance and update back to database
  entry.Amount = entry.Amount - 50.000
  err = global_db.C("bank").UpdateId(entry.Id, entry)

  if err != nil{
    panic("update error")
  }
  count_withdraw += 1
  fmt.Printf("count_withdraw %d\n", count_withdraw)

  io.WriteString(w, "ok")

}

func main() {
  session, _ := mgo.Dial("localhost:27017")
  fmt.Printf("Session is %p\n", session)
  global_db = session.DB( "db_log" )

  //make sure it is empty first
  global_db.C("bank").DropCollection()

  //Init amount is 1000 USD
  user := Currency{Account : "tuanbach", Amount: 1000.00, Code:"USD"}
  err := global_db.C("bank").Insert(&user)

  if err != nil{
    panic("insert error")
  }

  http.HandleFunc("/", withdraw)
  http.ListenAndServe(":8000", nil)
}
```

Trong code này, user tên "tuanbach" có 1000usd trong tài khoản. Mỗi lần anh ta rút được 50usb, vậy tổng số lần rút đúng là **20** (1000/50) lần. Tuy nhiên nếu tôi dùng ab test với command:
```
ab -n 5000 -c 1000 http://localhost:8000/
```

Tool này sẽ gọi tới server 5000 request và có 1000 request cùng xảy ra cùng 1 lúc. Nhìn vào console log, tôi tính anh này rút được tới **1947** lần, rất lớn so với số lần rút thưc tế là 20 phải không ?

Giờ hãy khắc phục bằng cách thêm transaction kỳ diệu vào,  sửa code như sau:
```go
//Add sync to import
import "sync"
//Define mu
var mu = &sync.Mutex{}

func withdraw(w http.ResponseWriter, r *http.Request) {
  entry := Currency{}
  //step 1: get current amount

  //Solution here, Lock other thread access this section of code until it's unlocked
  mu.Lock()
  defer mu.Unlock()
  err := global_db.C("bank").Find(bson.M{"account":  "tuanbach"}).One(&entry)

  if err != nil {
    panic(err)
  }

  fmt.Printf("%+v\n", entry)
  //step 2: check if balance is valid to widthdraw
  if entry.Amount < 50.00 {
    fmt.Printf("out_of_balance %d\n", count_withdraw)
    io.WriteString(w, "out_of_balance")
    return
  }

  //step 3: subtract current balance and update back to database
  entry.Amount = entry.Amount - 50.000
  err = global_db.C("bank").UpdateId(entry.Id, entry)

  if err != nil{
    panic("update error")
  }
  count_withdraw += 1
  fmt.Printf("count_withdraw %d\n", count_withdraw)

  io.WriteString(w, "ok")

}
```

Đoạn code này không khác mấy so với code trước đó :), Tôi chỉ khai báo biến mu ( kiểu Mutex ) và thêm dòng
```go
mu.Lock()
defer mu.Unlock()
```
Trước bước lấy số dư hiện tại và thao tác cập nhật lại số dư. Tôi có dòng code
```go
mu.Lock
```
Dòng này sẽ khóa chỉ cho phép 1 request được truy cập, các request khác phải chờ khi đoạn code được mở khóa

```go
 defer mu.Unlock()
```
Dòng tiếp theo, đây là bước mở khóa. defer là từ khóa trong Golang, nó sẽ thực thi khi các dòng code bên dưới xử lý xong thì sẽ được mở khóa.

**Kết quả**: xem console log thì tôi thấy user này chỉ rút được 20 lần, đây là điều chúng ta muốn. Golang giải quyết bài toán khá đơn giản nhưng nếu bạn quay lại các ngôn ngữ lập trình khác thì quả là một bài toán khó.

Giờ hãy sử dụng 1 cách khác, Tôi sẽ dùng hàng đợi( Queue ), Mỗi khi có yêu cầu rút tiền tối sẽ đưa vào hàng đợi, hàng đợi này chỉ có 1 tiến trình xử lý nên vẫn đảm bảo được xử lý tuần tự và cách này có thể áp dụng với các ngôn ngữ lập trình khác.  Tuy nhiên, nội dung cho phần này khá dài nên tôi sẽ viết bài khác để có thể hiểu rõ hơn.