---
layout: post
title:  "Download nhiều file cùng lúc với Golang."
date:   2015-10-04 18:34:10 +0700
categories: [golang]
---
Có một tính năng rất hay ở Golang đó là quản lý tiến trình song song, nó có thể tạo ra nhiều tiến trình cùng xử lý 1 công việc nào đó cùng lúc. Tôi sẽ lợi dụng tính năng này để làm 1 tool download nhiều file cùng 1 lúc như vậy thời gian chờ để tải hết danh sách file sẽ nhanh hơn là tải luân phiên từng file.

Trước khi đi vào code thì chúng ta cùng xem kết quả về thời gian hoàn thành là như thế nào qua bảng bên dưới.

Danh sách file cần tải về là 13 files với tổng cộng dung lượng là 92.5MB.

**Workers**

**Thời gian hoàn thành**

| Workers | Thời gian hoàn thành |
|---------|----------------------|
| 1       | 1m48.534732736s      |
| 2       | 1m7.218691043s       |
| 3       | 1m5.696948058s       |
| 4       | 46.680558785s        |
| 5       | 38.562380383s        |
| 6       | 31.869491434s        |
| 7       | 29.963840247s        |
| 8       | 31.866479283s        |



Bảng kết quả này chỉ là tương tối vì tôi chỉ test 1 trường hợp 1 lần nhưng cũng cho thấy nếu worker càng nhiều thì thời gian hoàn thành càng nhanh, khi đạt tới ngưỡng 8 worker thì  không còn hiệu quả nữa,  điều này cũng dễ hiểu vì nếu worker đạt tới 1 mức độ nào đó thì băng thông internet của bạn sẽ full nên càng nhiều cũng không giải quyết được.

Thử so sánh thời gian tải file sử dụng 1 worker là 1phút48 giây và 7 worker với 29.9 giây thì ta thấy thời gian dùng worker hiệu quả sẽ nhanh hơn gấp 3 vì thế cũng đáng để dùng phải không nào :)

### Code

Đây là file main của tôi.
```go
package main 

import (
  "log"
  "fmt"
  "os"
  "time"
  "tuanbach/helpers"
  "sync"
)

func main() {

  t1 := time.Now()

  var wg sync.WaitGroup
  wg.Add(1)
  list_files := []string{
    "http://www.sample-videos.com/video/mp4/720/big_buck_bunny_720p_5mb.mp4",
    "https://s.yimg.com/uy/build/images/sohp/inspiration/night-sky3.jpg",
    "https://s.yimg.com/uy/build/images/sohp/hero/carlin-gill-valley3.jpg",
    "https://farm1.staticflickr.com/564/21309678523_c023623e2d_o_d.jpg",
    "https://farm1.staticflickr.com/710/21490453750_18514dbc02_o_d.jpg",
    "http://s3.amazonaws.com/TimeScapes/images/stills/4k/big_sur.jpg",
    "http://www.hdwallpapers.in/walls/katniss_the_hunger_games_mockingjay_part_2-wide.jpg",
    "https://farm6.staticflickr.com/5832/21794430722_d08e786a17_o_d.jpg",
    "http://www.hdwallpapers.in/walls/oblivion_movie-wide.jpg",
    "https://farm1.staticflickr.com/664/21506735842_65fcbe880f_o_d.jpg",
    "https://farm6.staticflickr.com/5646/21223463020_19d4993c3e_o_d.jpg",
    "http://www.hdwallpapers.in/download/hot_pursuit_2015_movie-3840x2160.jpg",
    "https://farm1.staticflickr.com/693/20645722103_59d6387f5c_o_d.jpg"}

  log.Printf("List files %v %d\n", list_files, len(list_files))

  c := make(chan string)
  count_done := 0

  //workers are 7
  for i:=0; i< 7;i++{

    go func ( c chan string ) {

      for {
        select{
          case url := <-c:
            log.Printf("url is %s\n", url)

            fileName :=  helpers.FileNameFromUrl( url )
            
            relative_path, err_download := helpers.DownloadMedia( "/home/geek/Desktop", "test_download", url, fileName )
            count_done = count_done + 1
            if ( err_download != nil ){
              fmt.Printf("error when download file %v \n", err_download)
            }else{
              fmt.Printf("Download file %d is ok %s \n", count_done, relative_path)
            }


            if count_done == len(list_files){

              t2 := time.Now()
              fmt.Printf("%d files is downloaded  with total time is  %v\n", count_done,t2.Sub(t1) )
              
              wg.Done()
            }
        }
      }

    }(c)
  }

  go func () {
    for _, url := range list_files{
      c <- url
    }    
  }()


  wg.Wait()
  
}
```

Code hiện tại tôi đang thiết lập là 7 với channel c là unlimited size, nghĩa là có bấy nhiêu file thì bỏ vào bấy nhiêu rồi chờ worker làm việc. Bạn có thể thiết lập size của channel bằng cách thêm tham số  ví dụ:
```go
c := make(chan string,10)
```

Lệnh này tôi tạo channel với size là 10. lúc ban đầu thì 10 file đầu tiên sẽ vào channel, khi worker nào đó tải xong thì file khác mới được đưa vào channel, và worker tiếp tục lấy ra và tiếp tục đến khi hết file.

Trong file main có sử dụng thư viện "tuanbach/helpers" và đây là code của file trong thư viện, đây là file chứa các hàm để tải file về, xử lý lưu vào ổ cứng, ...
```go
package helpers

import (
  "fmt"
  "io"
  "net/http"
  "os"
  "strings"
  "path"
)

func Join( arr_path ...string ) string{

  var result string = "";

  for _, value := range arr_path{
    result = path.Join( result, value )
  }

  return result

}

func Exists(path string) (bool, error) {
  _, err := os.Stat(path)
  if err == nil { return true, nil }
  if os.IsNotExist(err) { return false, nil }
  return true, err
}


func SubPath( file_path string, folder  string)  string{
  return strings.Replace( file_path, folder, "", -1 )
}

/**
 * Get file extension
 */
func Ext( file_path string,)  string{
  return strings.Replace( path.Ext( file_path ), ".", "", -1 ) 
}

func FileNameFromUrl( url string ) string {
  tokens := strings.Split(url, "/")
  fileName := tokens[len(tokens)-1]
  return fileName
}

func DownloadFromUrl(url string, fileName string) error {

  result, _ := Exists( fileName )

  if result == true{
    fmt.Printf("File %s is exists\n", fileName)
    return nil
  }

  folderDir := path.Dir( fileName )
  result, _ = Exists( folderDir )
  if result != true {
    fmt.Println("Create folder " + folderDir)
    os.MkdirAll( folderDir, 0755 )
  }

  fmt.Println("Downloading", url, "to", fileName)

  // TODO: check file existence first with io.IsExist
  output, err := os.Create(fileName)
  if err != nil {
    fmt.Println("Error while creating", fileName, "-", err)
    return  err
  }
  defer output.Close()

  response, err := http.Get(url)
  if err != nil {
    fmt.Println("Error while downloading", url, "-", err)
    return err
  }
  defer response.Body.Close()

  n, err := io.Copy(output, response.Body)
  if err != nil {
    fmt.Println("Error while downloading", url, "-", err)
    return err
  }

  fmt.Println(n, "bytes downloaded.")
  return  nil
}


func DownloadMedia( uploadPath string, subFolder string, filePath string, fileName string) (string, error) {

  fullPath := Join( uploadPath, subFolder , fileName )

  fmt.Println( "Full path is " + fullPath )

  err := DownloadFromUrl( filePath, fullPath )

  if err != nil{
    return "", err
  }else{
    return SubPath( fullPath, uploadPath ), err
  }
    
}
```
Vì đây là code nằm trong dự án mình đang làm nhưng  vẫn chưa xong nên chưa comment code đầy đủ được. mình sẽ update comment khi nào rãnh.