# Tổng quan 

n8n là một nền tảng tự động hóa quy trình làm việc mã nguồn mở, cho phép bạn kết nối và tự động hóa các ứng dụng, dịch vụ web thông qua giao diện trực quan, không yêu cầu kỹ năng lập trình chuyên sâu. Công cụ này giúp chuyển đổi các tác vụ thủ công, lặp lại thành các luồng công việc tự động hiệu quả. Bài viết này sẽ hướng dẫn các bạn cách sử dụng N8N để quét một hoặc nhiều trang WordPress mong muốn (Bài viết này sẽ lấy trang vietnix.vn làm ví dụ)

Để quét các trang lớn như vietnix.vn, cần phải chia thành nhiều Workflow khác nhau để giảm lượng dữ liệu ghi vào các node quá nhiều dẫn đến các lỗi như crash Node hoặc "Invalid String Length"

# Các Workflow

## 1. Workflow điều hướng

Workflow này sẽ thực hiện các bước sau


![image](https://github.com/user-attachments/assets/c3d748cb-f965-4277-855d-11a6b54ecb0e)

Quy trình thực hiện mô hình sẽ bao gồm 6 bước chính với các node cụ thể

1. Bước 1 - Lên lịch để thực hiện kiểm tra định kì

Node 1: Cron Jobs - Đây là 1 trong những node trigger của n8n được sử dụng để trigger workflow chạy theo định kì được cấu hình

2. Bước 2 - Lấy tất cả Domain

Node 1: Google Sheets - Sử dụng node này để lấy tất cả Domain tron Google Sheets 

Node 2: Code - Tạo ra đoạn mã JavaScript để parse tất cả Domain trong Google Sheets thành dạng code sử dụng cho Workflow

3. Bước 3 - Kiểm tra trạng thái kết nối các Domain và kiểm tra sự tồn tại của site XML

Node 1: HTTP Request - Gửi các request đến các website và kiểm tra phản hồi, nếu trạng thái status lỗi sẽ thông báo về Discord channel "Status", nếu không có lỗi sẽ thực hiện node kế

Node 2: HTTP Request - Gửi các request để kiểm tra xem tính năng để xem website XML đã bật chưa, nếu chưa thực hiện báo về Discord channel "Status", nếu không có lỗi sang bước 4

Node 3: Discord - Gửi thông báo qua message được trả về từ node Code

4. Lấy toàn bộ tên miền con từ website gốc

Node 1: Code - Sử dụng Code JavaScript để tách toàn bộ các site Sub XML từ kết quả trả về của node HTTP Request trước

Node 2: HTTP Request - Gửi request đến toàn bộ site sub XML để kiểm tra trạng thái truy cập và lấy tất cả Domain trong đó

Node 3: Code - Sử dụng code JavaScript để lấy toàn bộ domain từ sub XML (Cũng là toàn bộ Domain của website)

5. Tách Images và IFrames

Node 1: Loop Over Items - Sử dụng node này để tách tất cả giá trị domain lấy từ bước trước ra để chạy kiểm tra, thay vì kiểm tra tổng thể (khi lỗi sẽ khó xác định site lỗi)

Node 2: HTTP Request - Gửi request về domain này để đảm bảo tính truy cập, nếu có error status sẽ gửi thông báo Discord channel "Status" 

Node 3: HTML - Sử dụng 2 Node này để tách và lấy toàn bộ images và iframes từ kết quả trả về của node trước. Kết quả tra về sẽ là 2 mảng có giá trị images và iframes

Node 4: Node Code - Lọc các Data URI từ image và thêm đường dẫn http: cho các website chưa có 

Node 5: IF - Sử dụng node điều kiện để kiểm tra rỗng, nếu cả 2 mảng đều rỗng thì quay lại vòng lặp, nếu không sẽ thực hiện node tiếp theo 

Node 6 - Node 7: IF - Sử dụng node điều kiện để kiểm tra trống Images và IFrame tránh việc quét báo lỗi, nếu mảng trống thì bỏ qua, nếu mảng tồn tại thì đến bước kiểm tra 

Node 8: Discord - Gửi thông báo nếu Node 2 có website lỗi 

Node 9 - Node 10: Split Out - Dùng 2 node này để tách mảng ra thành từng thành phần images và iframes tránh việc xử lý quá tải ở bước tiếp theo

6. Kiểm tra và thông báo 

Node 1 - Node 2: HTTP Request - Sử dụng 2 node này để request đến hình ảnh và iframe của trang web

Node 3: IF - Sử dụng node điều kiện để kiểm tra nếu tồn tại lỗi sẽ gửi thông báo về Discord, nếu không sẽ quay lại vòng lặp

Node 4: Discord - Sử dụng để gửi thông báo lỗi tới channel Images, IFrames

# Các bước thực hiện

## 1. Lên lịch kiểm tra website

### Node 1: Cron Jobs - Đây là 1 trong những node trigger của n8n được sử dụng để trigger workflow chạy theo định kì được cấu hình

Để tạo được 1 lịch kiểm tra website định kì, chúng ta sẽ sử dụng node “Cron Jobs” thực hiện trigger định kì dựa theo cấu hình. Đây là 1 trong những node có khả năng trigger để Workflow chạy. Để trigger định kì 1 tiếng kiểm tra định kì 1 lần, thực hiện cấu hình trong node như sau. Trong đó

- Mode: Chế độ - chúng ta sẽ chọn Every X để điền theo ý muốn

- Value: Giá trị thời gian mong muốn

- Unit: Đơn vị thời gian (Giờ, phút, giây,...)
  
<div align='center'>

![uog_Image_2](https://github.com/user-attachments/assets/f8e61a68-21fc-4aa5-b239-484e5553d8e4)

</div>

## 2. Lấy và tách các Domain để thực hiện Workflow

### Hình ảnh tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/6e907f0e-6c51-4e60-b2b7-d34e42e4c066)

</div>

### Node 1: Google Sheets - Sử dụng node này để lấy tất cả Domain tron Google Sheets 

Trước tiên, thực hiện lấy danh sách các Domain cần kiểm tra bằng cách sử dụng Google Sheet chứa danh sách và in ra danh sách đó. Để thực hiện trên Workflow, sử dụng Credentials Google Sheet Account, bạn sẽ thực hiên cấu hình tại hình bút chì bao gồm các thông tin về API được tạo trên GCP. Nếu bạn chưa biết cách kết nối, hãy xem tạo [https://vietnix.vn/ket-noi-n8n-den-google-cloud-apis/](https://vietnix.vn/ket-noi-n8n-den-google-cloud-apis/)

<div align='center'>
  
![zrt_Image_3](https://github.com/user-attachments/assets/0dcc25f9-e216-41c6-8283-4af3044cdf22)

</div>

### Node 2: Code - Tạo ra đoạn mã JavaScript để parse tất cả Domain trong Google Sheets thành dạng code sử dụng cho Workflow

Sau đó, thực hiện parse dữ liệu text này để sử dụng cho node split, bạn có thể sử dụng node code và thêm code sau. Mục đích của code này để đọc danh sách domain từ output của một node khác (Read Domain File) và biến từng dòng thành một item JSON riêng biệt.

```
return items.map(item => ({
  json: {
    domain: item.json['Domain '],
  }
}));
```

<div align='center'>
  
![il6_Image_4](https://github.com/user-attachments/assets/f61b7c6d-9635-4236-9348-4390c4fc2682)

</div>

## 3. Kiểm tra trạng thái website và trang XML 

### Hình ảnh tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/edf0b902-a85c-47ca-ae6f-f3a47d2d7ce0)

</div>

### Node 1: HTTP Request - Gửi các request đến các website và kiểm tra phản hồi, nếu trạng thái status lỗi sẽ thông báo về Discord channel "Status", nếu không có lỗi sẽ thực hiện node kế

Trước khi kiểm tra Images và IFrames cần phải kiểm tra xem có truy cập được trang WordPress không. thêm 1 node HTTP Request để gửi yêu cầu HTTP đến website và trang XML. Nếu website có thể truy cập, thực hiện bước xuất ảnh và iFrame, nếu không, thực hiện thông báo vào Discord tại channel “Status”.

Cấu hình node HTTP đó như sau. Lưu ý các trường sau:

- Request Method: Chọn “POST” để kiểm tra Website thay vì "GET" để tránh lỗi "Many redirects"

- URL: Bạn có thể thực hiện kéo thả từ domain ở bước Split Domain vào

- Full Response: Nên bật để lấy thông tin Status và Error

- Thực hiện tương tự với node của XML 

<div align='center'>

![image](https://github.com/user-attachments/assets/61cfc046-daac-4669-b79d-0eb07a218234)

</div> 

### Node 2: HTTP Request - Gửi các request để kiểm tra xem tính năng để xem website XML đã bật chưa, nếu chưa thực hiện báo về Discord channel "Status", nếu không có lỗi sang bước 4

<div align="center">

![image](https://github.com/user-attachments/assets/4179c96b-4844-4540-b6af-a60efc1d478d)

</div>

### Node 3: Discord - Gửi thông báo qua message được trả về từ node Code

Phần Message của node Discord sẽ cấu hình như ảnh bên dưới. Lưu ý

+ {{ $('Extract Main Domains').item.json.domain }}: Lấy phần domain từ node "Extract Main Domains"
+ {{ $json.error.code }}: Lấy Error Code từ node trước
+ {{ $json.error.status }}: Lấy Error Status từ node trước

<div align="center">

![image](https://github.com/user-attachments/assets/464df7d0-7818-4d5e-8c1b-865d3acb172a)

</div>

## 4. Lấy toàn bộ Sub URL 

### Hình ảnh tổng quan

<div align="center">

![image](https://github.com/user-attachments/assets/fcd3fd56-814d-4e5e-b939-b2bb0bdb05f3)

</div>

### Node 1: Code - Sử dụng Code JavaScript để tách toàn bộ các site Sub XML từ kết quả trả về của node HTTP Request trước

Sử dụng node code để lọc ra toàn bộ các subXML, thêm code JavaScript dưới đây. Khi này node sẽ trả về giá trị là các subpage

```
const xml = items[0].json.body || items[0].json.data || '';
if (typeof xml !== 'string') {
  throw new Error('Sitemap XML is not a valid string');
}

const urls = [...xml.matchAll(/<loc>(.*?)<\/loc>/g)].map(match => match[1]);

return urls.map(url => ({
  json: {
    subpage: url,
  },
}));
```

### Node 2: HTTP Request - Gửi request đến toàn bộ site sub XML để kiểm tra trạng thái truy cập và lấy tất cả Domain trong đó

Sử dụng `{{ $json.subpage }}` để lấy toàn bộ giá trị sub XML được liệt kê ra từ node trước

<div align="center">

![image](https://github.com/user-attachments/assets/6c988051-ce58-4327-8bae-d1bd29b152f9)

</div>

### Node 3: Code - Sử dụng code JavaScript để lấy toàn bộ domain từ sub XML (Cũng là toàn bộ Domain của website)

Thực hiện lấy các subpage đó để kiểm tra qua node HTTP Request và tách các URL cần kiểm tra bằng node code. Thực hiện code JavaScript sau

```
const results = [];

for (const item of items) {
  const xml = item.json.body || item.json.data || '';

  if (typeof xml !== 'string') {
    throw new Error('Sitemap XML is not a valid string');
  }

  const matches = [...xml.matchAll(/<loc>(.*?)<\/loc>/g)];
  const urls = matches.map(match => match[1]);

  for (const domain of urls) {
    results.push({ json: { domain } });
  }
}

return results;
```

## 5. Kiểm tra Images và IFrames

### Hình ảnh tổng quan 

<div align='center'>
  
![image](https://github.com/user-attachments/assets/22657da4-e7c9-4c24-905e-1ecda612611a)

</div>

### Node 1: Loop Over Items - Sử dụng node này để tách tất cả giá trị domain lấy từ bước trước ra để chạy kiểm tra, thay vì kiểm tra tổng thể (khi lỗi sẽ khó xác định site lỗi)

<div align="center">

![image](https://github.com/user-attachments/assets/1f416f4e-4224-499f-838f-d2652cdb393c)

</div>

### Node 2: HTTP Request - Gửi request về domain này để đảm bảo tính truy cập, nếu có error status sẽ gửi thông báo Discord channel "Status" 

Lưu ý

Trường `{{ $json.domain }}` để lấy Domain từ node trước

<div align="center">
  
![image](https://github.com/user-attachments/assets/f31f1fb9-fb47-48e9-9d03-ea7f909b56a4)

</div>

### Node 3 - Node 4: HTML - Sử dụng 2 Node này để tách và lấy toàn bộ images và iframes từ kết quả trả về của node trước. Kết quả tra về sẽ là 2 mảng có giá trị images và iframes

Lọc ra tất cả Images và IFrame từ trang web thành 2 mảng tương ứng. Thực hiện bằng cách sử dụng node HTML với lựa chọn “Extract HTML Content” và cấu hình như sau để lấy ảnh và iFrames.

<div align='center'>
  
![Screenshot from 2025-06-16 14-53-19](https://github.com/user-attachments/assets/a13b8488-f3ba-4e55-93d4-ff77052a1a7e)

</div>

### Node 4: Node Code - Lọc Data URI và thêm đường dẫn cho các image chưa có

Sử dụng code sau

```
return items.map(item => {
  const images = item.json.image || [];

  const filtered = images
    .filter(src => typeof src === 'string' && !src.startsWith('data:')) // loại bỏ base64
    .map(src => {
      if (src.startsWith('//')) {
        return 'https:' + src;
      }
      if (!src.startsWith('http')) {
        return 'https://' + src;
      }
      return src;
    });

  return {
    json: {
      ...item.json,
      image: filtered,
    },
  };
});
```

<div align='center'>

![image](https://github.com/user-attachments/assets/988d3044-da24-40eb-87d6-9dad5964fe07)

</div>

### Node 5: IF - Sử dụng node điều kiện để kiểm tra rỗng, nếu cả 2 mảng đều rỗng thì quay lại vòng lặp, nếu không sẽ thực hiện node tiếp theo 

Để tránh trường hợp Website không có hình ảnh và không có iFrame để kiểm tra, thêm node IF với điều kiện các mảng trống, nếu các mảng trống thì sẽ chạy nhánh “True” quay về vòng lặp thực hiện kiểm tra website tiếp theo. Nếu 1 trong 2 mảng có phần tử thì sẽ chạy nhanh false để kiểm tra. Các trường json bạn có thể kéo thả từ bước gộp trước. Lưu ý, chon kiểu dữ liệu là mảng để tránh lỗi.

<div align='center'>

![image](https://github.com/user-attachments/assets/16580317-a154-4125-bcc2-af957cfeb7bd)

</div>

### Node 6 - Node 7: IF - Sử dụng node điều kiện để kiểm tra trống Images và IFrame tránh việc quét báo lỗi, nếu mảng trống thì bỏ qua, nếu mảng tồn tại thì đến bước kiểm tra 

Tại nhánh false, thêm 2 node điều kiện để kiểm tra 1 trong 2 mảng trống, mảng nào trống sẽ bỏ qua không check tránh lỗi không mong muốn

<div align='center'>
  
![image](https://github.com/user-attachments/assets/ef5d1f5c-7020-4c49-81b0-bbd75f258d0c)

![image](https://github.com/user-attachments/assets/38252912-fe3e-46a9-afc7-99ef41099021)

</div>

### Node 8: Discord - Gửi thông báo nếu Node 2 có website lỗi 

Tại Message của Node Discord, cấu hình như sau. Lưu ý

+ Domain: {{ $json.domain }}: Lấy Domain từ node trước
+ Status: {{ $json.error.status }}: Lấy trạng thái error từ node trước
+ Log: {{ $json.error.code }}: Lấy code error từ node trước

```
Domain: {{ $json.domain }}
Status: {{ $json.error.status }}
Log: {{ $json.error.code }}
////////////////////////////////
```

<div align="center">

![Screenshot from 2025-06-15 20-36-41](https://github.com/user-attachments/assets/32d4bc55-310c-4c21-a303-be86bea1b49e)

</div>

### Node 9 - Node 10: Split Out - Dùng 2 node này để tách mảng ra thành từng thành phần images và iframes tránh việc xử lý quá tải ở bước tiếp theo

Cấu hình node Split Out kéo 2 mảng images và iframe vào 2 node tương ứng để tách xử lý từng dữ liệu trong mảng

<div align="center">

![image](https://github.com/user-attachments/assets/57baf271-fb5e-4b7c-babd-e3a713c75048)

</div>

## 6. Kiểm tra và gửi thông báo

### Hình ảnh tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/e4ae4d1b-b3c9-4538-bd50-0cb5b8f4e281)

</div>

### Node 1 - Node 2: HTTP Request - Sử dụng 2 node này để request đến hình ảnh và iframe của trang web

Thêm 2 node HTTP Request để tiến hành duyệt qua các Images và IFrames có trong mảng. Nếu không có lỗi sẽ quay lại vòng lặp, nếu có lỗi sẽ thông báo ngay lập tức về Discord

<div align='center'>

![image](https://github.com/user-attachments/assets/8ab49825-1013-42af-8b45-2f55a2ef329b)


![image](https://github.com/user-attachments/assets/808e5c2a-6c3c-48ac-9f55-818e283eca31)

</div>

### Node 3: IF - Sử dụng node điều kiện để kiểm tra nếu tồn tại lỗi sẽ gửi thông báo về Discord, nếu không sẽ quay lại vòng lặp

Thêm 1 node IF để check status.

<div align="center">

![image](https://github.com/user-attachments/assets/fab31575-d39b-4a18-95c2-4ead5dd14b13)

</div>

### Node 4: Discord - Sử dụng để gửi thông báo lỗi tới channel Images, IFrames
Cuối cùng thưc hiện thêm node Discord để gửi thông báo, bạn có thể lấy Webhook và thêm vào Credentials Discord. Gửi message về channel Images IFrame với format như sau.

Lưu ý:

+ {{ $('Loop Over Items').item.json.domain }}: Lấy giá trị Domain từ node vòng lặp
+ ❌ Error Code: {{ $('Check Error').item.json.error.code }}: Lấy error code từ node "Check Error"
+ ❌ Error Status: {{ $('Check Error').item.json.error.status }}: Lấy error status từ node "Check Error"
+ 📍 Destination: {{ $json.error.input }}: Lấy input(File lỗi) từ node trước

```

🚨 Domain Error on Images IFrames Report

{{ $('Loop Over Items').item.json.domain }}
❌ Error Code: {{ $('Check Error').item.json.error.code }}
❌ Error Status: {{ $('Check Error').item.json.error.status }}
📍 Destination: {{ $json.error.input }}
!!!!!
```

<div align="center">

![image](https://github.com/user-attachments/assets/fda22c8e-3924-4a31-8693-153e49454053)

</div>

## Demo 

### 1. Trang web không thể truy cập

Sừa Domain trên Google Sheet (Thêm /111 đằng sau Domain) và chạy Workflow

Workflow sẽ chạy từ node "Fetch Domain" để quét Domain sang node Discord để thông báo Status và cuối cùng quay lại vòng lặp để chạy các Domain khác. Đối với Domain lỗi sẽ không được chạy nhánh True vào vòng lặp 

<div align="center">

![th1](https://github.com/user-attachments/assets/5155e8ed-cc5d-4465-b13e-07c210bddb4f)

</div>

Thông báo lỗi

<div align="center">

![image](https://github.com/user-attachments/assets/97fc38a1-1151-499e-9994-0e6b882ae614)

</div>

### 2. Trang lỗi site XML 

Thực hiện tắt site bằng lệnh `a2dissite` và chạy Workflow

Workflow sẽ chạy từ node "Get Sitemap XML" ra nhánh false và tiến hành thông báo về Discord, sau đó quay lại vòng lặp để quét Domain khác

<div align="center">

![455262620-abed42dc-e624-43df-89c4-a30602a74159](https://github.com/user-attachments/assets/1d81525a-3f90-450e-b0d3-71c1fd9099ac)

</div>

Thông báo lỗi 

<div align="center"

![image](https://github.com/user-attachments/assets/417132ca-6437-47be-9006-5816367cee49)

</div>

### 3. Lỗi Images IFrames bất kì

Thêm đường dẫn lỗi ảnh vào site 2 và thực hiện test Workflow

Workflow sẽ chạy đúng quy trình tất cả các bước và đi vào ra nhánh True tại Node IF cuối cùng. Sau khi thông báo sẽ quay lại vòng lặp quét domain tiếp theo

<div align="center">

![image](https://github.com/user-attachments/assets/232d8871-60e1-4a15-baec-9123dc3ce385)

</div>

Thông báo lỗi 

<div align="center">

![image](https://github.com/user-attachments/assets/d04a78ea-826f-4b5d-a725-b6f866784130)

</div>

### 4. Tất cả website chạy bình thường

Workflow khi này sẽ không chạy vào các nhánh báo lỗi nữa

<div align="center">

![image](https://github.com/user-attachments/assets/50fbe89d-3d41-4844-a86d-35befd6304a4)

<div>
