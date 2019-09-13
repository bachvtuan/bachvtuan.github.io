---
layout: post
title:  "Tự tạo ví tiền với Bitcoin."
date:   2016-06-11 10:34:10 +0700
categories: [nodejs, bitcoin]
---
Bitcoin là gì ? Mọi người quan tâm thì google để hiểu căn bản đã nhé. Hôm nay tôi  chỉ hướng dẫn cách tạo ví tiền rồi tự quản lý tiền trong đó thôi. Bạn không cần phải sử dụng ví tiền thứ 3 như localbitcoin,paxful hay blockchain để quản lý.


Vì sao phải cần thiết tự tạo ví tiền trong khi các dịch vụ kia có thể tạo giùm mình ?

Cách đây mấy ngày có thông tin sàn giao dịch Bitfinex bị mất gần 66 triệu USD tiền mặt tương đương 119,756 BTC. Nếu bạn có tài khoản ở đây thì bạn sẽ nghĩ gì, làm gì ? Hệ thống Bitcoin không dễ bị hack nhưng ở đây là vấn đề bảo mật thông tin của cách dịch vụ wallet để cho thông tin user bị lộ và kèm theo private_key cũng có thể bị hack nên users bị mất toàn bộ số tiền. Và buồn thay bạn có thể thấy số tiền của mình đi đên tài khoản nào đó nhưng không cách nào truy ra ai là chủ nhân thật sự. Ví dụ tôi có [link](https://blockchain.info/address/1L2JsXHPMYuAa9ugvHGLwkdstCPUDemNCf) , đây là link giao dịch tiền bị hack với giá trị lên tới 19,943 BTC, bạn chỉ có thể xem mà không thể biết số tiền cuối cùng sẽ về đâu vì nó chuyển từ tài khoản này sang tài khoản kia như một móc xích .

Vậy nếu bạn không tin tưởng dịch vụ nào hết thì hãy tự tạo cho mình tài khoản và tự quản lý nó. Khi bạn tự tạo tài khoản thì bạn sẽ giữ private_key và public_key . Public_key chính là địa chỉ mà người khác có thể gửi tiền tới cho bạn, và quan trọng là private_key chỉ mình bạn biết, nếu bạn không tiết lộ thông tin này mà vẫn bị hack thì thôi chắc chẳng ai thèm dùng Bitcoin làm gì nữa. 🙂

Cách làm
Mình sẽ sử dụng nodejs để làm:

Bạn sẽ cần cài module “bitcore-lib”.

``
npm install bitcore-lib
``

Đoạn code bên dưới sẽ generate ra private key và pubic key ngẫu nhiên:
```javascript
var bitcore = require("bitcore-lib");

var privateKey = new bitcore.PrivateKey();
console.log("privateKey",privateKey.toWIF())

var address = privateKey.toAddress();
console.log("publicKey",address.toString());
```

Khi bạn chạy lệnh này xong thì output sẽ có dạng:

```
privateKey L4RChqheNebkA6hoNEkkA2qkUtTqQDd9MRHvgC6JuDUNJt3vSRW5
publicKey 14hREFw2yh8xXKa68X6gC8iD1AZXgu2X57
```
Vậy là bạn đã có cho mình một tài khoản bitcoin 0 BTC. Ban phải cất giữ privateKey cẩn thận hoặc ghi ra giấy rồi cất nơi nào đó.

publicKey bạn cần ghi lại nhưng không cần phải bảo mật, che giấu gì và người ta có cũng  chả làm gì được nhé.

Giờ định dang của bạn khi đi giao dịch trên Bitcoin sẽ là địa chỉ publicKey, nếu người khác cần chuyển tiền cho bạn thì bạn gửi địa chỉ đó cho họ.

Giờ bạn có thể kiểm tra tài khoản của mình với link https://blockchain.info/address/{publicKey}. Nếu sử dụng key trên thì link sẽ là

[link](https://blockchain.info/address/14hREFw2yh8xXKa68X6gC8iD1AZXgu2X57) Bạn sẽ thấy Balance là 0 BTC vì tài khỏan này chưa có giao dịch gì cả.

Vậy làm thế nào để chuyển tiền cho người khác ? Bạn có thể sử dung tiếp thư viện trên để thưc hiện, nhưng tôi sẽ không đề cập ở đây. Nếu có thời gian tôi sẽ viết thêm.

**Ưu điểm:**
Vì mình bạn giữ privateKey nên sẽ không lo bị hack tài khoản, đêm dài có thể ngủ ngon.

**Nhược điểm:**
Nếu muốn thực hiện các giao dịch thì phải dùng code, điều này sẽ gây khó khăn cho những ai ít biết về lập trình.