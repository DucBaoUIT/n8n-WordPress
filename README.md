# Tổng quan 

n8n là một nền tảng tự động hóa quy trình làm việc mã nguồn mở, cho phép bạn kết nối và tự động hóa các ứng dụng, dịch vụ web thông qua giao diện trực quan, không yêu cầu kỹ năng lập trình chuyên sâu. Công cụ này giúp chuyển đổi các tác vụ thủ công, lặp lại thành các luồng công việc tự động hiệu quả

Mô hình dưới đây sẽ hướng dẫn các bạn cách tạo workflow N8N để kiểm tra website WordPress có bị lỗi images và iframes hay không.

Mô hình tổng quan

![image](https://github.com/user-attachments/assets/d1bd5c80-bd1e-4ec5-b13e-e542734cdd26)


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

</div>

Tiếp theo, sử dụng node loop với batch size là 1 để đảm bảo chạy workflow với 1 website mỗi lần


## 3. Kiểm tra trạng thái website và trang XML 

Trước khi kiểm tra Images và IFrames cần phải kiểm tra xem có truy cập được trang WordPress không. thêm 1 node HTTP Request để gửi yêu cầu HTTP đến website và trang XML. Nếu website có thể truy cập, thực hiện bước xuất ảnh và iFrame, nếu không, thực hiện thông báo vào Discord tại channel “Status”.

<div align='center'>
  
![image](https://github.com/user-attachments/assets/7001b1ed-d577-48bb-acac-51564c5dc24e)

</div> 

Cấu hình node HTTP đó như sau. Lưu ý các trường sau:

- Request Method: Chọn “GET” để kiểm tra Website

- URL: Bạn có thể thực hiện kéo thả từ domain ở bước Split Domain vào

- Full Response: Nên bật để lấy thông tin Status và Error

- Thực hiện tương tự với node của XML 

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
## 5. Kiểm tra Images và IFrames

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

Tại nhánh false, thực hiện kiểm tra bằng node HTTP Request giống như cách kiểm tra website từ trước đó. Cấu hình cho kiểm tra image như sau, thực hiện tương tự cho iframe.

<div align='center'>
  
![6os_Image_11](https://github.com/user-attachments/assets/fd6e6bd7-a982-47f7-a722-e7a79b3178c1)

</div>

Cuối cùng, sau khi kiểm tra, sử dụng IF node để lọc thông báo, nếu có 1 request bị error sẽ chạy nhánh false gửi thông báo về discord. Nếu cả 2 đều trả về status 200 thì sẽ chạy nhánh true quay lại vòng lặp

## 6. Thông báo

Trước khi thông báo, chúng ta sẽ gộp tất cả lỗi lại qua node code để thông báo 1 lần thay vì từng tệp lỗi. Lưu ý, code sau đây sẽ thêm điều kiện để kiểm tra file trống, nếu file trống nó sẽ là cờ để thực hiện điều kiện IF sau để tránh thông báo lỗi vì file trống, Thực hiện code sau

```
const results = [];
let hasEmptyInput = false;

for (const item of items) {
  const code = item.json.error?.code;
  const input = item.json.error?.input;

  if (!input || input.trim() === '') {
    hasEmptyInput = true;
    break;
  }

  if (code) {
    results.push(`❌ Error Code: ${code}\n📍 Destination: ${input}`);
  }
}

return [
  {
    json: {
      message: results.length > 0 ? `🚨 Domain Error Report\n\n${results.join('\n\n')}` : '',
      hasEmptyInput,
    },
  },
];
```
Cuối cùng thưc hiện thêm node Discord để gửi thông báo, bạn có thể lấy Webhook và thêm vào Credentials Discord. Gửi message về channel Images IFrame với format như sau. Lưu ý

{{ $('Split Domains').item.json.domain }} : Là domain của website bị lỗi

{{ $json.message }}: Là message sau khi được xử lý qua Code Node

<div align='center'>

![9bf_Image_12](https://github.com/user-attachments/assets/23547d66-b427-4691-a75f-5f0eb5aab6a4)

</div>

## 7. Thông báo Status Web lỗi

Cấu hình thông báo lỗi như sau. Chú thích

- Domain: {{ $json.domain }} - Là Domain của Website bị lỗi

- Status: {{ $json.error.status }} - Trạng thái lỗi của Domain

- Log: {{ $json.error.code }} - Cụ thể về lỗi

<div align='center'>

![xn3_Image_13](https://github.com/user-attachments/assets/bd21318a-fc1f-4d78-b27d-83d507750606)

</div>


