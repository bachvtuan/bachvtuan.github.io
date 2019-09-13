---
layout: post
title:  "Golang: Transaction cho Mongodb ( Sử dụng channel )"
date:   2016-04-09 18:34:10 +0700
categories: [golang, mongodb]
---
Ở trong bài viết tôi đã hướng dẫn cách lock code trong golang để gỉa lập transaction cho mongodb. Trong bài này tôi sẽ hướng dẫn cách khác mà tôi nghĩ hiệu năng sẽ tốt hơn đó là sử dụng Channel trong golang.

Channel là tính năng rất mạnh trong Golang, Channel đơn giản hóa chỉ là hàng đợi(Queue) . Khi ta đưa thông tin vào hàng đợi thì sẽ có thread lấy dữ liệu đó ra lần lượt rồi xử lý.  Data nào được đưa vào trước thì sẽ được xử lý trước.

Giả sử tôi lần lượt đưa vào hàng đợi các số lần lượt 3,6,2,8. Thì số 3 sẽ xử lý trước rồi đến 6,2,8.

Tôi sẽ lợi dụng điều này để giả lập transaction cho mongodb với ý tưởng là khi ai đó có yêu cầu rút tiền tôi sẽ đưa yêu cầu này vào channel rồi nó sẽ được xử lý tuần tự.

Để có tính thuyết phục hơn bài trước, tôi sẽ tạo 100 users, mỗi user sẽ có 1000$. Nếu đúng thi tối đa tổng số lần rút tiền của hệ thống là (1000$/50) \* 100 = 2000 lần. ( Mỗi lần rút 50$ ).

Bạn có thể xem full code ở file [queue_code.go](https://github.com/bachvtuan/Golang-Mongodb-Transaction-Example/blob/master/queue_code.go)

Trong code này tôi tạo 2 hàng đợi "in" và "out".
```go
in = make(chan string)
out = make(chan Result)
```

Tiếp theo tôi tạo ngẫu nhiên 100 user với mỗi tài khoản có 1000$.
```go
for i := 1; i <= maxUser; i++ {
user := Currency{ Account : "user" + strconv.Itoa( i ) , Amount: 1000.00, Code:"USD" }
err := global_db.C("bank").Insert(&user)

if err != nil{
  panic("insert error")
}          
}
```
Channel "in" sẽ tiếp nhận yêu cầu rút tiền từ user nào đó, sau đó  tôi khai báo 1 listener cho "in"  để xử lý yêu cầu rút tiền:
```go
go func ( in *chan string ) {
  for {
    select{
      case account := <-*in:
        /*count_queue += 1
        fmt.Printf("count_queue %d\n", count_queue)*/
        entry := Currency{}
        err := global_db.C("bank").Find(bson.M{"account":  account }).One(&entry)

        //time.Sleep(100 * time.Millisecond)

        if err != nil {
          panic(err)
        }

        fmt.Printf("%+v\n", entry)
        //step 2: check if balance is valid to widthdraw
        if entry.Amount < 50.00 {
          fmt.Printf("out_of_balance\n")
          out <-  Result{ Account: account, Result: "out_of_balance"}
          //io.WriteString(w, "out_of_balance")
          
        }else{
          //step 3: subtract current balance and update back to database
          entry.Amount = entry.Amount - 50.000
          err = global_db.C("bank").UpdateId(entry.Id, entry)

          if err != nil{
            panic("update error")
            out <-  Result{ Account: account, Result: "update error"}
          }
          countWithdraw += 1
          fmt.Printf("countWithdraw %d\n", countWithdraw)
          out <-  Result{ Account: account, Result: fmt.Sprintf("countWithdraw %d\n", countWithdraw)}
        }
    }
  }

}(&in)
```
Bạn có thể thấy sau khi xử lý rút tiền xong ở listener này thì tôi sẽ đưa kết quả vào channel "out"
```
out <-  Result{ Account: account, Result: fmt.Sprintf("countWithdraw %d\\n", countWithdraw)}
```
Đây là trường hợp thành công
```
out <-  Result{ Account: account, Result: "out_of_balance"}
```
Còn đây là trường hợp hết tiền không rút được.

Vấn đề còn lại là đăng ký listener khác trong request rút tiền để lấy ra kết quả ở trên.
```go
for {
  select {
  case result := <- out:

    if result.Account == account{
      fmt.Printf("Result %s and countWithdraw is %d\\n", result.Result, countWithdraw)
      io.WriteString(w, result.Result)
      wg.Done()
      //should return, otherwise it's still pop out value from out channel
      return          
    }else{
      fmt.Printf("Dismatch: %s and %s\\n", result.Account, account)
      panic("why ?, Something went wrong")
      //push to out again
      //out <- result
    }

  };
}
```
Bạn có thể test bằng cách : chạy code trên và sử dụng benmarch ab bằng lệnh

ab -n 5000 -c 1000 http://localhost:8000/

Cách tốt hơn ?
--------------

Bạn để ý là cách trên chỉ có 1 channel để xử lý như vậy thì nếu hệ thống có quá nhiều yêu cầu rút tiền thì channel này sẽ có thể quá tải và không phát huy hết hiệu năng của mongodb vì mỗi lần chỉ có 1 request tới mongodb . Tôi muốn tạo ra nhiều channel để xử lý nhằm rút ngắn thời gian nhưng vẫn đảm bảo được khả năng giả lập transaction.

Bạn có thể xem full code ở file [multiple_queue_code.go](https://github.com/bachvtuan/Golang-Mongodb-Transaction-Example/blob/master/multiple_queue_code.go).

Và đây là một số thay đổi so với cách trên.

Tôi khai báo "in" và "out" là danh sách mảng chứa các channel.
```go
var in []chan string
var out []chan Result
```
Rồi cấp phát bộ nhớ, maxThread ở đây tôi khai báo là 10, như vậy sẽ có 10 channel xử lý song song.
```go
in = make([]chan string, maxThread)
out = make([]chan Result, maxThread)
```
Mỗi channel trong mảng "in" tôi cần có listener để xử lý khi có data vào:
```go
for i := range in {

  go func ( subIn *chan string, index int ) {

    for {
      select{
        case account := <-*subIn:
          fmt.Printf("On worker %d \n", index + 1)
          /*count_queue += 1
          fmt.Printf("count_queue %d\n", count_queue)*/
          entry := Currency{}
          err := global_db.C("bank").Find(bson.M{"account":  account }).One(&entry)
          
          //time.Sleep(100 * time.Millisecond)

          if err != nil {
            panic(err)
          }

          //fmt.Printf("%+v\n", entry)
          //step 2: check if balance is valid to widthdraw
          if entry.Amount < 50.00 {
            //fmt.Printf("out_of_balance\n")
            out[ index ] <-  Result{ Account: account, Result: "out_of_balance"}
            //io.WriteString(w, "out_of_balance")
            
          }else{
            //step 3: subtract current balance and update back to database
            entry.Amount = entry.Amount - 50.00
            err = global_db.C("bank").UpdateId(entry.Id, entry)

            if err != nil{
              //panic("update error")
              out[ index ] <-  Result{ Account: account, Result: "update error"}
            }
            
            //mu.Lock()
            countWithdraw = countWithdraw + 1
            
            fmt.Printf("countWithdraw %d\n", countWithdraw)
            
            //mu.Unlock()

            out[ index ] <-  Result{ Account: account, Result: fmt.Sprintf("countWithdraw %d\n", countWithdraw)}
          }
      }
    }

  }(&in[i], i)
}
```
Giờ vấn đề tiếp theo là chọn user nào vào channel nào cho phù hợp để đảm bảo toàn vẹn dữ liệu:
```go
number := Random( 1, maxUser )

//Allocate to appropriate channel number based on number by get the last number in the random number.
channelNumber :=  number % maxThread
account := "user" + strconv.Itoa( number )
```
Mỗi lần rút tiền tôi tạo id user ngẫu nhiên đây chính là "number".  Tôi lấy "number" chia cho maxThread(10) và lấy số dư. Số dư chính là index trong channel "in" sẽ được xử lý. Đó là luật của tôi để đảm bảo tất cả channel đều có xác suất nhận lượng dữ liệu giống nhau và mỗi user luôn luôn được xử lý bởi 1 channel duy nhất.

Khi một channel trong "in" giả sử với index là "i" xử lý xong tôi sẽ gửi dữ liệu vào channel "out" với index là "i".

Và cuối cùng, tôi cung đăng ký listener cho channel với index "i" nằm trong "out".
```go
  for {
    select {
    case result := <- out:

      if result.Account == account{
        fmt.Printf("Result %s and countWithdraw is %d\n", result.Result, countWithdraw)
        io.WriteString(w, result.Result)
        wg.Done()
        //should return, otherwise it's still pop out value from out channel
        return          
      }else{
        fmt.Printf("Dismatch: %s and %s\n", result.Account, account)
        panic("why ?, Something went wrong")
        //push to out again
        //out <- result
      }

    };
  }
```
Giờ hãy so sánh thời gian xử lý 2 cách :

### Benchmark 1

Đầu tiên tôi muôn test 10000 request và mỗi lần sẽ có 1000 request gọi cùng 1 lúc
```
ab -n 10000 -c 1000 http://localhost:8000
```
**queue_code.go( cách 1 )**
```
Concurrency Level: 1000
Time taken for tests: 2.678 seconds
Complete requests: 10000
Failed requests: 9991
(Connect: 0, Receive: 0, Length: 9991, Exceptions: 0)
Total transferred: 1318893 bytes
HTML transferred: 148893 bytes
Requests per second: 3733.55 \[#/sec\] (mean)
Time per request: 267.842 \[ms\] (mean)
Time per request: 0.268 \[ms\] (mean, across all concurrent requests)
Transfer rate: 480.87 \[Kbytes/sec\] received
```
**multiple_queue_code.go( cách 2  )**
```
Concurrency Level:      1000
Time taken for tests:   1.224 seconds
Complete requests:      10000
Failed requests:        9991
   (Connect: 0, Receive: 0, Length: 9991, Exceptions: 0)
Total transferred:      1318894 bytes
HTML transferred:       148894 bytes
Requests per second:    8172.20 \[#/sec\] (mean)
Time per request:       122.366 \[ms\] (mean)
Time per request:       0.122 \[ms\] (mean, across all concurrent requests)
Transfer rate:          1052.56 \[Kbytes/sec\] received
```
Như vậy cách 2 nhanh hơn gấp 2 lần so với cách 1.

### Benchmark 2

Giờ hãy nâng lên 2000 requests gọi cùng 1 lúc
```
ab -n 10000 -c 2000 http://localhost:8000/
```
**queue_code.go( cách 1 )**

ab gọi được 9000 requests thì server không phản hồi được vì chỉ dùng 1 hàng đợi nên bị quá tải ( FAILED )

**multiple_queue_code.go( cách 2  )**
```
Concurrency Level:      2000
Time taken for tests:   1.258 seconds
Complete requests:      10000
Failed requests:        9991
   (Connect: 0, Receive: 0, Length: 9991, Exceptions: 0)
Total transferred:      1318893 bytes
HTML transferred:       148893 bytes
Requests per second:    7948.40 \[#/sec\] (mean)
Time per request:       251.623 \[ms\] (mean)
Time per request:       0.126 \[ms\] (mean, across all concurrent requests)
Transfer rate:          1023.74 \[Kbytes/sec\] received
```
Server vẫn hoạt đông bình thường, thậm chí thời gian phản hồi chỉ chậm hơn chút so với benmarch 1.

Trường hợp chuyển tiền:
-----------------------

Nãy giờ ta chỉ làm rút tiền cho 1 user nào đó, giờ này thử transaction với trường hợp khó hơn đó là chuyển tiền giữa các user. Vì số tiền chỉ transfer qua về giữa các user trong hệ thông nên sau khi thực hiên benchmark xong thì tổng số tiền vẫn giống tổng số tiền trước khi gọi.

Bạn có thể lấy tổng số tiền băng lệnh trong mongodb:
```javascript
db.getCollection('bank').aggregate(
   [
     {
       $group:
         {
           _id: null,
           totalAmount: { $sum: "$amount" },
           count: { $sum: 1 }
         }
     }
   ]
)
```
hệ thông test cho 100 user mỗi user có 1000$ nên tổng tiền sẽ là 100.000$.

Full code của ví dụ này bạn có thể xem ở file [multiple_queue_transfer.go](https://github.com/bachvtuan/Golang-Mongodb-Transaction-Example/blob/master/multiple_queue_transfer.go).

Có một lưu ý ở phần này để đảm bảo tính toàn vẹn là khi cập nhât lại số tiền tôi không dùng cách cũ nữa
```go
entry.Amount = entry.Amount - 50.00
err = global_db.C("bank").UpdateId(entry.Id, entry)
```
mà sẽ dùng
```go
change := bson.M{"$inc": bson.M{"amount": -50 }}
```
Cho bên A khi gửi qua B

Và đây là code khi bên B nhận được
```go
change = bson.M{"$inc": bson.M{"amount": 50 }}
```
Đố bạn vì sao đấy ?