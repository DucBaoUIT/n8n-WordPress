# Tổng quan 

n8n là một nền tảng tự động hóa quy trình làm việc mã nguồn mở, cho phép bạn kết nối và tự động hóa các ứng dụng, dịch vụ web thông qua giao diện trực quan, không yêu cầu kỹ năng lập trình chuyên sâu. Công cụ này giúp chuyển đổi các tác vụ thủ công, lặp lại thành các luồng công việc tự động hiệu quả

Mô hình dưới đây sẽ hướng dẫn các bạn cách tạo workflow N8N để kiểm tra website WordPress và các website con có bị lỗi images và iframes hay không.

Mô hình tổng quan

![image](https://github.com/user-attachments/assets/0de8e530-b9cb-4857-bcd1-4dc4afcb5d53)

Quy trình thực hiện mô hình sẽ bao gồm 6 bước chính với các node cụ thể

1. Bước 1 - Lên lịch để thực hiện kiểm tra định kì

Node 1: Cron Jobs - Đây là 1 trong những node trigger của n8n được sử dụng để trigger workflow chạy theo định kì được cấu hình

2. Bước 2 - Lấy tất cả Domain

Node 1: Google Sheets - Sử dụng node này để lấy tất cả Domain tron Google Sheets 

Node 2: Code - Tạo ra đoạn mã JavaScript để parse tất cả Domain trong Google Sheets thành dạng code sử dụng cho Workflow

3. Bước 3 - Kiểm tra trạng thái kết nối các Domain và kiểm tra sự tồn tại của site XML

Node 1: HTTP Request - Gửi các request đến các website và kiểm tra phản hồi, nếu trạng thái status lỗi sẽ thông báo về Discord channel "Status", nếu không có lỗi sẽ thực hiện node kế

Node 2: HTTP Request - Gửi các request để kiểm tra xem tính năng để xem website XML đã bật chưa, nếu chưa thực hiện báo về Discord channel "Status", nếu không có lỗi sang bước 4

Node 3: Code - Thực hiện phân loại lỗi của website (Node 1) hay lỗi của XML (Node 2) để trả về thông báo 

Node 4: Discord - Gửi thông báo qua message được trả về từ node Code

4. Lấy toàn bộ tên miền con từ website gốc

Node 1: Code - Sử dụng Code JavaScript để tách toàn bộ các site Sub XML từ kết quả trả về của node HTTP Request trước

Node 2: HTTP Request - Gửi request đến toàn bộ site sub XML để kiểm tra trạng thái truy cập và lấy tất cả Domain trong đó

Node 3: Code - Sử dụng code JavaScript để lấy toàn bộ domain từ sub XML (Cũng là toàn bộ Domain của website)

5. Tách Images và IFrames

Node 1: Loop Over Items - Sử dụng node này để tách tất cả giá trị domain lấy từ bước trước ra để chạy kiểm tra, thay vì kiểm tra tổng thể (khi lỗi sẽ khó xác định site lỗi)

Node 2: HTTP Request - Gửi request về domain này để đảm bảo tính truy cập, nếu có error status sẽ gửi thông báo Discord channel "Status" 

Node 3 - Node 4: HTML - Sử dụng 2 Node này để tách và lấy toàn bộ images và iframes từ kết quả trả về của node trước. Kết quả tra về sẽ là 2 mảng có giá trị images và iframes

Node 5: Node Merge - Gộp 2 mảng lại 

Node 6: IF - Sử dụng node điều kiện để kiểm tra rỗng, nếu cả 2 mảng đều rỗng thì quay lại vòng lặp, nếu không sẽ thực hiện node tiếp theo 

Node 7 - Node 8: IF - Sử dụng node điều kiện để kiểm tra trống Images và IFrame tránh việc quét báo lỗi, nếu mảng trống thì bỏ qua, nếu mảng tồn tại thì đến bước kiểm tra 

6. Kiểm tra và thông báo 

Node 1 - Node 2: HTTP Request - Sử dụng 2 node này để request đến hình ảnh và iframe của trang web

Node 3: IF - Sử dụng node điều kiện để kiểm tra nếu tồn tại lỗi sẽ gửi thông báo về Discord, nếu không sẽ quay lại vòng lặp

Node 4: Discord - Sử dụng để gửi thông báo lỗi tới channel Images, IFrames

# Các bước thực hiện

## 1. Lên lịch kiểm tra website

Để tạo được 1 lịch kiểm tra website định kì, chúng ta sẽ sử dụng node “Cron Jobs” thực hiện trigger định kì dựa theo cấu hình. Đây là 1 trong những node có khả năng trigger để Workflow chạy. Để trigger định kì 1 tiếng kiểm tra định kì 1 lần, thực hiện cấu hình trong node như sau. Trong đó

- Mode: Chế độ - chúng ta sẽ chọn Every X để điền theo ý muốn

- Value: Giá trị thời gian mong muốn

- Unit: Đơn vị thời gian (Giờ, phút, giây,...)
  
<div align='center'>

![uog_Image_2](https://github.com/user-attachments/assets/f8e61a68-21fc-4aa5-b239-484e5553d8e4)

</div>

## 2. Lấy và tách các Domain để thực hiện Workflow

Trước tiên, thực hiện lấy danh sách các Domain cần kiểm tra bằng cách sử dụng Google Sheet chứa danh sách và in ra danh sách đó. Để thực hiện trên Workflow, sử dụng Credentials Google Sheet Account, bạn sẽ thực hiên cấu hình tại hình bút chì bao gồm các thông tin về API được tạo trên GCP. Nếu bạn chưa biết cách kết nối, hãy xem tạo [https://vietnix.vn/ket-noi-n8n-den-google-cloud-apis/](https://vietnix.vn/ket-noi-n8n-den-google-cloud-apis/)

<div align='center'>
  
![zrt_Image_3](https://github.com/user-attachments/assets/0dcc25f9-e216-41c6-8283-4af3044cdf22)

</div>

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

![image](https://github.com/user-attachments/assets/6e907f0e-6c51-4e60-b2b7-d34e42e4c066)

</div>


## 3. Kiểm tra trạng thái website và trang XML 

Trước khi kiểm tra Images và IFrames cần phải kiểm tra xem có truy cập được trang WordPress không. thêm 1 node HTTP Request để gửi yêu cầu HTTP đến website và trang XML. Nếu website có thể truy cập, thực hiện bước xuất ảnh và iFrame, nếu không, thực hiện thông báo vào Discord tại channel “Status”.

Cấu hình node HTTP đó như sau. Lưu ý các trường sau:

- Request Method: Chọn “GET” để kiểm tra Website

- URL: Bạn có thể thực hiện kéo thả từ domain ở bước Split Domain vào

- Full Response: Nên bật để lấy thông tin Status và Error

- Thực hiện tương tự với node của XML 

<div align='center'>
  
![image](https://github.com/user-attachments/assets/edf0b902-a85c-47ca-ae6f-f3a47d2d7ce0)

</div> 

## 4. Lấy toàn bộ Sub URL 

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

Sau đó, kiểm tra tất cả URL này, nếu URL tồn tại và có thể truy cập, thực hiện bước tiếp theo, nếu có lỗi, gửi thông báo về discord
<div align='center'>
  
![image](https://github.com/user-attachments/assets/38c59d57-6211-4730-9f58-13fbd854c8c2)

</div>

## 5. Kiểm tra Images và IFrames

Sử dụng Loop với Batch Size là 1 để tạo vòng lặp quét qua từng URL và lấy Images, IFrames bên trong (Nếu URL lỗi lập tức gửi thông báo)

Lọc ra tất cả Images và IFrame từ trang web thành 2 mảng tương ứng. Thực hiện bằng cách sử dụng node HTML với lựa chọn “Extract HTML Content” và cấu hình như sau để lấy ảnh và iFrames.

<div align='center'>
  
![IjB_Image_7](https://github.com/user-attachments/assets/7262a27f-21e3-4c89-995e-fdac151532cf)

![FoL_Image_8](https://github.com/user-attachments/assets/0c03b150-82c9-4120-826d-ef4705784ef9)

</div>

Sau đó, dùng node merge thực hiện gộp tất cả lại dưới dạng SQL. Nếu bạn sử dụng 2 input thì sử dụng lệnh mặc định của node vẫn chạy được. Thu được output như sau:

<div align='center'>

![LoF_Image_9](https://github.com/user-attachments/assets/b91c79ed-6058-4836-b5d0-b1644707df65)

</div>

Để tránh trường hợp Website không có hình ảnh và không có iFrame để kiểm tra, thêm node IF với điều kiện các mảng trống, nếu các mảng trống thì sẽ chạy nhánh “True” quay về vòng lặp thực hiện kiểm tra website tiếp theo. Nếu 1 trong 2 mảng có phần tử thì sẽ chạy nhanh false để kiểm tra. Các trường json bạn có thể kéo thả từ bước gộp trước. Lưu ý, chon kiểu dữ liệu là mảng để tránh lỗi.

<div align='center'>

![8cX_Image_10](https://github.com/user-attachments/assets/7fd9c23f-32c1-4e66-940e-165fc00af235)

</div>

Tại nhánh false, thêm 2 node điều kiện để kiểm tra 1 trong 2 mảng trống, mảng nào trống sẽ bỏ qua không check tránh lỗi không mong muốn

<div align='center'>
  
![image](https://github.com/user-attachments/assets/ef5d1f5c-7020-4c49-81b0-bbd75f258d0c)

![image](https://github.com/user-attachments/assets/38252912-fe3e-46a9-afc7-99ef41099021)

</div>

<div align='center'>
  
![image](https://github.com/user-attachments/assets/1d884d19-e30f-4a9d-bdea-277b9fbf4f7c)

</div>


## 6. Kiểm tra và gửi thông báo

Thêm 2 node HTTP Request để tiến hành duyệt qua các Images và IFrames có trong mảng. Nếu không có lỗi sẽ quay lại vòng lặp, nếu có lỗi sẽ thông báo ngay lập tức về Discord

<div align='center'>

![image](https://github.com/user-attachments/assets/8ab49825-1013-42af-8b45-2f55a2ef329b)


![image](https://github.com/user-attachments/assets/808e5c2a-6c3c-48ac-9f55-818e283eca31)

</div>

Thêm 1 node IF để check status. Cuối cùng thưc hiện thêm node Discord để gửi thông báo, bạn có thể lấy Webhook và thêm vào Credentials Discord. Gửi message về channel Images IFrame với format như sau

```
🚨 Domain Error Report

{{ $('Loop Over Items').item.json.domain }}
❌ Error Code: {{ $json.error.code }}
❌ Error Status: {{ $json.error.status }}
📍 Destination: 
!!!!!
```

## 7. Thông báo Status Web lỗi

Cấu hình thông báo lỗi như sau. Chú thích

- Domain: {{ $json.domain }} - Là Domain của Website bị lỗi

- Status: {{ $json.error.status }} - Trạng thái lỗi của Domain

- Log: {{ $json.error.code }} - Cụ thể về lỗi

<div align='center'>

![xn3_Image_13](https://github.com/user-attachments/assets/bd21318a-fc1f-4d78-b27d-83d507750606)

</div>


