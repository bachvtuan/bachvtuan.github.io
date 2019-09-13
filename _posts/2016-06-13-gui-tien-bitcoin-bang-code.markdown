---
layout: post
title:  "Gửi tiền bitcoin bằng code"
date:   2016-06-13 18:34:10 +0700
categories: [nodejs, bitcoin]
---
Trong bài trước mình có hướng dẫn cách tự tạo ví tiền, giờ mình sẽ hướng dẫn cách dùng ví để "xài tiền", mình có thể tự gửi tiền cho các địa chỉ khác mà không qua dịch vụ thứ 3, làm được bước này thì xem như là bạn cũng gần hoàn chỉnh quản lý bitcoin của mình rồi. Điều quan trọng là bạn không sợ bi hack vì private key vẫn chỉ mình bạn biết.

Để chạy code này bạn cần setup nodejs và các thư viện liên quan.


Nodejs v4.4.3( or higher )

    "devDependencies": {
        "bitcore-lib": "^0.13.18",
        "async": "0.9.0",
        "request": "2.53.0"
    }

Đây là full code , bạn có thể xem ở link này  [https://gist.github.com/bachvtuan/a1fb6c745f2c6e56a57d1c4dbdede5a7](https://gist.github.com/bachvtuan/a1fb6c745f2c6e56a57d1c4dbdede5a7)
```javascript 
/**
 * package.json

  "devDependencies": {
    "bitcore-lib": "^0.13.18",
    "async": "0.9.0",
    "request": "2.53.0"
  }
*/

var bitcore = require("bitcore-lib");
var fs = require("fs");
var request = require('request');
var Async = require("async");

/** END EXAMPLE INPUT, YOU SHOULD MODIFY TO YOUR CASE **/
// Your public key
var from_public = "your_public_key_here";
// Your private key
var private_key = 'your_private_key_here';

// Define which addresses you wan to send, just 1 item or more, below are 2 example addresses
var send_addresses = [
    //adress and amount in satoshies
    { address: "1Hyxrc88gd6s9w1MpVz3e1NLf4FTZxBby6", amount: 327880 },//approximate 2$
    { address: "19tqEocN86Y4rHyeB2G4ezUpYNxv94sdmM", amount: 163940 }, //approximate 1$
];


/** END EXAMPLE INPUT, YOU SHOULD MODIFY TO YOUR CASE **/


const push_bitcoin_url = "https://blockchain.info/pushtx";
const check_address_url = "https://api.blockcypher.com/v1/btc/main/addrs/{address}?unspentOnly=1";
//According https://api.blockcypher.com/v1/btc/main
const fee_per_kb = 71050; // 71050 satoshies 0.43$/kb fee
const min_transaction_amount = 546; // request dust is 546 satoshies

Async.waterfall([


    function (callback) {

        var check_url = check_address_url.replace('{address}', from_public);
        console.log("check_url", check_url);

        request(check_url, function (error, response, body) {
            if (!error && response.statusCode == 200) {
                //console.log(body);
                var json = JSON.parse(body);

                var unspend_transactions = [];

                if (json.unconfirmed_txrefs) {
                    for (var i = 0; i < json.unconfirmed_txrefs.length; i++) {
                        var item = json.unconfirmed_txrefs[i];

                        if (!item.double_spend) {
                            //if there is unconfirmation transaction but there is my public address here
                            if (item.address == from_public) {
                                unspend_transactions.push(item);
                            }
                        }
                        else {
                            console.log("detected double spend on unconfirmation transactions");
                        }
                    }
                }

                if (json.txrefs) {
                    for (var i = 0; i < json.txrefs.length; i++) {
                        var item = json.txrefs[i];

                        if (!item.double_spend) {
                            unspend_transactions.push(item);

                        } else {
                            console.log("detected double spend on confirmed transactions");
                        }
                    }
                }

                callback(null, unspend_transactions);

            } else {
                callback("can not get input address information");
            }
        })

    },
    function (input_transactions, callback) {


        //console.log("input_transactions",input_transactions);


        var script = new bitcore.Script(new bitcore.Address(from_public)).toHex();

        var privateKey = new bitcore.PrivateKey(private_key);

        var total_value = 0;

        var transaction = new bitcore.Transaction();

        for (var i = 0; i < input_transactions.length; i++) {
            var item = input_transactions[i];
            var utxo = {
                "txId": item.tx_hash,
                "outputIndex": item.tx_output_n,
                "address": from_public,
                "script": script,
                "satoshis": item.value
            };

            total_value += item.value;

            transaction.from(utxo)

        }

        //Total output amount by satoshies

        var total_output = 0;
        for (var i = 0; i < send_addresses.length; i++) {
            var address_item = send_addresses[i];
            transaction.to(address_item.address, address_item.amount)
            total_output += address_item.amount;
        }

        //  estimate size of transaction and calculate fee
        var fee = Math.floor(fee_per_kb * transaction._estimateSize() / 1024);

        console.log("total_output", total_output);
        console.log("total_value", total_value);
        console.log("fee", fee);

        if (total_value < total_output + fee) {
            return callback("Not enough for create transaction");
        }

        var change_amount = total_value - total_output - fee;

        if (change_amount > 0 && change_amount < min_transaction_amount) {
            return callback("The change amount is too small");
        }

        console.log("change_amount", change_amount);


        transaction.fee(fee);
        if (change_amount > 0) {
            transaction.change(from_public)
        }

        transaction.enableRBF()
            .sign(privateKey);

        var tx_hex = transaction.serialize();

        console.log("tx_hex", tx_hex);

        /*console.log( JSON.stringify(transaction.toObject()), "end");*/



        var post_params = {
            url: push_bitcoin_url, form: { tx: tx_hex }
        };

        request.post(post_params, function (error, response, body) {

            if (!error && response.statusCode == 200) {
                console.log("body", body) // Show the HTML for the Google homepage.
            } else {
                console.log("body", body) // Show the HTML for the Google homepage.
                console.log("statusCode", response.statusCode);
            }

            callback(null);

        });

    }
], function (err, result) {
    console.log('info', "END TASK");

    if (err) {
        console.log('info', err);
    }

    process.exit();
});     
```

Code thì khá dài nhưng để gửi được bitcoin, bạn cần quan tâm đoạn code trong phần mình comment input information
    
```javascript 
  // Your public key
  var from_public = "your_public_key_here";
  // Your private key
  var private_key = 'your_private_key_here';
  
  // Define which addresses you wan to send, just 1 item or more, below are 2 example addresses
  var send\_addresses = [
      //adress and amount in satoshies
      { address: "1Hyxrc88gd6s9w1MpVz3e1NLf4FTZxBby6", amount: 327880 },//approximate 2$
      { address: "19tqEocN86Y4rHyeB2G4ezUpYNxv94sdmM", amount: 163940 }, //approximate 1$
  ];
```
    
    
Ở đây bạn sửa lại public key và private key . Đồng thời nhập các địa chỉ cần gửi đi và số lượng bitcoin ( đơn vị satoshis ) .
Xong việc chỉ cần gọi "node your\_file\_code.js" là tiêu tiền được rồi :D
Nếu bạn sợ mình tạo transaction sẽ bị mất tiền, bạn có thể thêm dòng return dưới code "console.log("tx\_hex", tx\_hex);"

Rồi khi chạy code bạn copy đoạn tx\_hex được xuất ra và vào link này [https://blockchain.info/decode-tx](https://blockchain.info/decode-tx) . Bạn paste đoạn hex vào mà xem thử input và output có đúng không.

Good luck.