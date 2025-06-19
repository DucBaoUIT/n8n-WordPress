# Tổng quan 

n8n là một nền tảng tự động hóa quy trình làm việc mã nguồn mở, cho phép bạn kết nối và tự động hóa các ứng dụng, dịch vụ web thông qua giao diện trực quan, không yêu cầu kỹ năng lập trình chuyên sâu. Công cụ này giúp chuyển đổi các tác vụ thủ công, lặp lại thành các luồng công việc tự động hiệu quả. Bài viết này sẽ hướng dẫn các bạn cách sử dụng N8N để quét một hoặc nhiều trang WordPress mong muốn (Bài viết này sẽ lấy trang vietnix.vn làm ví dụ)

Để quét các trang lớn như vietnix.vn, cần phải chia thành nhiều Workflow khác nhau để giảm lượng dữ liệu ghi vào các node quá nhiều dẫn đến các lỗi như crash Node hoặc "Invalid String Length"

# Các Workflow

## A. Workflow điều hướng

### Mô hình tổng quan 

![image](https://github.com/user-attachments/assets/a105a717-5bd7-4d38-90fb-984bfdd4e1eb)

### 1. Lên lịch để kiểm tra website

##### Node 1: Cron Jobs - Đây là 1 trong những node trigger của n8n được sử dụng để trigger workflow chạy theo định kì được cấu hình

Để tạo được 1 lịch kiểm tra website định kì, chúng ta sẽ sử dụng node “Cron Jobs” thực hiện trigger định kì dựa theo cấu hình. Đây là 1 trong những node có khả năng trigger để Workflow chạy. Để trigger định kì 1 tiếng kiểm tra định kì 1 lần, thực hiện cấu hình trong node như sau. Trong đó

- Mode: Chế độ - chúng ta sẽ chọn Every X để điền theo ý muốn

- Value: Giá trị thời gian mong muốn

- Unit: Đơn vị thời gian (Giờ, phút, giây,...)
  
<div align='center'>

![uog_Image_2](https://github.com/user-attachments/assets/f8e61a68-21fc-4aa5-b239-484e5553d8e4)

</div>

### 2. Lấy và tách các Domain để thực hiện Workflow

#### Hình ảnh tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/6e907f0e-6c51-4e60-b2b7-d34e42e4c066)

</div>

#### Node 1: Google Sheets - Sử dụng node này để lấy tất cả Domain tron Google Sheets 

Trước tiên, thực hiện lấy danh sách các Domain cần kiểm tra bằng cách sử dụng Google Sheet chứa danh sách và in ra danh sách đó. Để thực hiện trên Workflow, sử dụng Credentials Google Sheet Account, bạn sẽ thực hiên cấu hình tại hình bút chì bao gồm các thông tin về API được tạo trên GCP. Nếu bạn chưa biết cách kết nối, hãy xem tạo [https://vietnix.vn/ket-noi-n8n-den-google-cloud-apis/](https://vietnix.vn/ket-noi-n8n-den-google-cloud-apis/)

<div align='center'>
  
![zrt_Image_3](https://github.com/user-attachments/assets/0dcc25f9-e216-41c6-8283-4af3044cdf22)

</div>

#### Node 2: Code - Tạo ra đoạn mã JavaScript để parse tất cả Domain trong Google Sheets thành dạng code sử dụng cho Workflow

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

### 3. Kiểm tra trạng thái website và trang XML 

#### Hình ảnh tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/e7172c6e-1e4c-450c-aaeb-9bb3bdff1011)

</div>

#### Node 1: HTTP Request - Gửi các request đến các website và kiểm tra phản hồi, nếu trạng thái status lỗi sẽ thông báo về Discord channel "Status", nếu không có lỗi sẽ thực hiện node kế

Trước khi kiểm tra Images và IFrames cần phải kiểm tra xem có truy cập được trang WordPress không. thêm 1 node HTTP Request để gửi yêu cầu HTTP đến website và trang XML. Nếu website có thể truy cập, thực hiện bước xuất ảnh và iFrame, nếu không, thực hiện thông báo vào Discord tại channel “Status”.

Cấu hình node HTTP đó như sau. Lưu ý các trường sau:

- Request Method: Chọn “POST” để kiểm tra Website thay vì "GET" để tránh lỗi "Many redirects"

- URL: Bạn có thể thực hiện kéo thả từ domain ở bước Split Domain vào

- Full Response: Tắt để tiết kiệm dữ liệu

- Thực hiện tương tự với node của XML 

<div align='center'>

![image](https://github.com/user-attachments/assets/082fe2b7-e938-421e-8562-bb42206bc9b9)

</div> 

#### Node 2: HTTP Request - Gửi các request để kiểm tra xem tính năng để xem website XML đã bật chưa, nếu chưa thực hiện báo về Discord channel "Status", nếu không có lỗi sang bước 4

<div align="center">

![image](https://github.com/user-attachments/assets/d58bb497-9059-48a0-a01d-0ee8934773a7)

</div>

#### Node 3: Discord - Gửi thông báo qua message được trả về từ node Code

Phần Message của node Discord sẽ cấu hình như ảnh bên dưới. Lưu ý

+ {{ $json.domain }}: Lấy phần domain từ node trước
+ {{ $json.error.code }}: Lấy Error Code từ node trước
+ {{ $json.error.status }}: Lấy Error Status từ node trước

<div align="center">

![image](https://github.com/user-attachments/assets/968092d6-d414-4213-8492-c84c1728a336)

</div>

#### Node 4: Discord - Gửi thông báo về việc site xml lỗi hoặc không tìm thấy 

Phần Message này khá giống với node 3. Chỉ thay đổi phần domain thành 

+ {{ $('Extract Main Domains').item.json.domain }}/sitemap.xml: để lấy domain từ node "Extract Main Domain"

<div align="center">

![image](https://github.com/user-attachments/assets/f18567ed-0b72-422f-89b9-ad5ee7688b84)

</div>

### 4. Lấy toàn bộ Sub URL 

#### Hình ảnh tổng quan

<div align="center">

![image](https://github.com/user-attachments/assets/fcd3fd56-814d-4e5e-b939-b2bb0bdb05f3)

</div>

#### Node 1: Code - Sử dụng Code JavaScript để tách toàn bộ các site Sub XML từ kết quả trả về của node HTTP Request trước

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

#### Node 2: HTTP Request - Gửi request đến toàn bộ site sub XML để kiểm tra trạng thái truy cập và lấy tất cả Domain trong đó

Sử dụng `{{ $json.subpage }}` để lấy toàn bộ giá trị sub XML được liệt kê ra từ node trước

<div align="center">

![image](https://github.com/user-attachments/assets/6c988051-ce58-4327-8bae-d1bd29b152f9)

</div>

#### Node 3: Code - Sử dụng code JavaScript để lấy toàn bộ domain từ sub XML (Cũng là toàn bộ Domain của website)

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

### 5. Loại bỏ domain trùng và đặt số thứ tự cho từng domain

#### Hình ảnh tổng quan

<div align="center">

![image](https://github.com/user-attachments/assets/a5f2525b-2444-44a8-ae5b-7312eddc12ad)

</div>

#### Node 1: Remove Duplicates - Sử dụng để loại bỏ các domain trùng 

Cài đặt node như hình dưới đây để lọc trùng domain. Lưu ý,

Mục "Operation" chọn "Remove Items Repeated Within Current Input" để lọc dữ liệu đầu vào

<div align="center">
  
![image](https://github.com/user-attachments/assets/8eacafbb-4bd0-4d96-845c-35994ec14593)

</div>

#### Node 2: Code - Sử dụng để đặt số thứ tự cho từng items

Để thêm số thứ tự cho từng Domains, sử dụng code JavaScript sau

```
return items.map((item, index) => {
  item.json._index = index;
  return item;
});
```

### 6. Tách Workflow để xử lý số lượng Domain

#### Hình ảnh tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/e5d17d08-3363-4489-87bb-17d5b4abdd2c)

</div>

#### Node 1: Split in Batches - Sử dụng Node này để phân chia số lượng Domain xử lý trong mỗi vòng lặp

Để quyết định số lượng domain chạy trong mỗi vòng lặp, cấu hình "Batch Size" trong node

<div align="center">

![image](https://github.com/user-attachments/assets/edc640e6-2ebf-4936-8ff8-5a4da7781757)

</div>

#### Node 2 - 3 - 4 - 5: IF - Sử dụng node này đặt điều kiện dựa trên index để đưa dữ liệu phù hợp vào các Workflow

Tại mỗi node IF, đặt điều kiện sao cho tổng số index là 1000 tương ứng với 1000 index đi vào đúng Workflow thích hợp (1000 Domain đầu tiên vào Workflow 1, và cứ thế tiếp diễn). Workflow 4 sẽ là Workflow chịu trách nhiệm quét tất cả các domain cuối cùng.

<div align="center">

![image](https://github.com/user-attachments/assets/7417885c-3e1e-4a8a-9874-6f9df785f4b8)

![image](https://github.com/user-attachments/assets/ce12dcb6-2518-4144-a9af-0e15d39f62d3)

![image](https://github.com/user-attachments/assets/15a06b74-c39e-4200-9a36-d5af951cc713)

![image](https://github.com/user-attachments/assets/a47d37ca-feee-40a8-90e6-638fa2783cdb)

</div>

#### Node 6 - 7 - 8 - 9: Execute Workflow - Sử dụng node này để gọi các Workflow khác hoạt động 

Cài dặt Node Excute Workflow như sau để gọi các Workflow khác. Lưu ý

+ Tại trường Workflow chọn "From list" để chọn từ các danh sách Workflow đã tạo

<div align="center">

![image](https://github.com/user-attachments/assets/a6351522-8928-4a60-9952-da19857e1d1f)

![image](https://github.com/user-attachments/assets/79193039-d0f5-4eef-9c6e-c515aa4fb4dc)

![image](https://github.com/user-attachments/assets/e2aaf745-c8ef-4119-9bdf-6ef87389cdcc)

![image](https://github.com/user-attachments/assets/09f1396f-97b0-441f-8fe5-f163220a5e02)

</div>

## B. Workflow thực thi phân tích hình ảnh (Cả 4 Workflow đều giống nhau)

### Mô hình tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/63595904-f123-4c7d-8246-500ea86a3112)

</div>

### 1. Kiểm tra trạng thái Website và lấy dữ liệu hình ảnh

#### Hình ảnh tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/8aa51a78-e2e4-4752-a6fc-cf5f105427fb)

</div>

#### Node 1: When Executed by Another Workflow - Node này sử dụng để trigger Workflow khi được gửi yêu cầu tới 

Tại mục "Input data mode", chọn "Accept All data" để chấp nhận mọi dữ liệu yêu cầu trigger workflow gửi tới.

<div align="center">

![image](https://github.com/user-attachments/assets/d4535c57-be7a-4401-90e9-2e6fa26b1915)

</div>

#### Node 2: Loop - Tách dữ liệu đầu vào thành nhiều vòng lặp để xử lý

Tại Loop của mỗi Workflow này, chúng ta sẽ chọn Batch size phù hợp để quét số lượng domain

<div align="center">

![image](https://github.com/user-attachments/assets/34171ce1-6c93-4392-b681-2395c33369a8)

</div>

#### Node 3: HTTP Request - Node này sử dụng để đảm bảo các doamin đầu vào khả dung và lấy data từ các domain

Lưu ý

Trường `{{ $json.domain }}` để lấy Domain từ node trước
Có thể tắt Full Response nếu muốn tránh dữ liệu quá nặng

<div align="center">
  
![image](https://github.com/user-attachments/assets/f31f1fb9-fb47-48e9-9d03-ea7f909b56a4)

</div>

#### Node 4: Discord - Gửi thông báo nếu Node 2 có website lỗi 

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

### 2. Xử lý hình ảnh

#### Hình ảnh tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/7444b90d-d914-485d-9f68-f4371b313f63)

</div>

#### Node 1: IF - Sử dụng node này để lọc mảng trống

Đặt điều kiện nếu mảng "Images" trả về từ node trước là trống (Trnag không có hình ảnh để quét) -> Chạy vòng lặp tiếp theo

<div align="center">

![image](https://github.com/user-attachments/assets/4a9f07e0-c5bb-476a-a1d3-fb71982b530e)

</div>

#### Node 2: Split Out - Sử dụng node này để tách toàn bộ image khỏi mảng 

Sử dụng Node Split out để tránh việc các node sau phải xử lý dữ liệu mảng quá lớn. Cài đặt "Field To Split Out" là "image" tương ứng với mảng trả về từ kết quả trước

<div align="center">

![image](https://github.com/user-attachments/assets/ae83fa00-43fc-43e6-9c0a-7da3a79251f1)

</div>

#### Node 3: Code - Lọc các hình ảnh qua code JS

Đảm bảo lọc các hình ảnh có giá trị trống và các ảnh "Data URI" trước khi thực hiện quét. Sử dụng code JS sau 

```
return items.filter(item => {
  const images = item.json.image;

  // Hàm kiểm tra URL hợp lệ, không phải Data URI
  const isValidUrl = (url) => {
    if (typeof url !== 'string') return false;
    const trimmed = url.trim();
    return (
      trimmed !== '' &&
      !trimmed.startsWith('data:') // Loại bỏ Data URI
    );
  };

  if (Array.isArray(images)) {
    const filtered = images.filter(isValidUrl);
    if (filtered.length === 0) return false;

    item.json.image = filtered; // Giữ lại mảng image đã lọc
    return true;
  }

  if (typeof images === 'string') {
    return isValidUrl(images);
  }

  return false;
});
```

Node 4: Remove Duplicates - Loại bỏ các ảnh có đường link trùng

Mục đích của việc loại bỏ trên để giảm thiểu lượng dữ liệu hình ảnh node tiếp theo phải quét

<div align="center">

![image](https://github.com/user-attachments/assets/ed195bba-527e-4484-a662-60b2297f4ebb)

</div>

### 3. Kiểm tra hình ảnh và thông báo 

#### Hình ảnh tổng quan 

<div align="center">

![image](https://github.com/user-attachments/assets/3ff872cc-8f96-414f-b38e-b2b0d5a16be0)

</div>

#### Node 1: HTTP Request - Kiểm tra các đường link hình ảnh trong website

Node này chỉ cần các giá trị trường HEAD để lấy được status của nó. Nếu quét lỗi sẽ chạy theo nhánh error qua node tiếp theo, nếu không sẽ sang vòng lặp tiếp theo

<div align="center">

![image](https://github.com/user-attachments/assets/ec9abb02-3695-4523-a488-4c25ae943f4e)

</div>

#### Node 2: Code - Gộp tất cả file lỗi của domain chứa nó để thông báo chung 1 lần

Thực hiện code JavaScript sau để gộp các lỗi lại

```
const domain = $('Loop Over Items').item?.json?.domain || 'Unknown';
const seen = new Set();
const results = [];

for (const item of items) {
  const error = item.json?.error;
  const statusCode = item.json?.error?.code || 'Unknown';
  const images = item.json?.image || [];

  for (const url of Array.isArray(images) ? images : [images]) {
    if (url && error && !seen.has(url)) {
      seen.add(url);
      results.push(`🔗URL: ${url} (💥Error: ${statusCode})`);
    }
  }
}

return [{
  json: {
    content: results.length
      ? `🚨Domain Error: ${domain}\n\n${results.slice(0, 50).join('\n')}`
      : '',
  },
}];
```

#### Node 3: Discord - Thông báo lỗi

Node Discord sẽ cấu hình lấy content từ node code đã gộp lỗi trước rồi thông báo qua Webhook

<div align="center">

![image](https://github.com/user-attachments/assets/4858fe93-b2d4-4582-8cf2-d0f14ce2dd56)

</div>

# Thực nghiệm 

## Domain lỗi (Thêm /111) đằng sau Domain

Quy trình Workflow

<div align="center">

![image](https://github.com/user-attachments/assets/95ab31d4-e97b-4dd5-8293-b2fc16905260)

</div>

Có thể thấy được Workflow sẽ thực thi nhánh false tại node quét website và gửi thông báo lỗi về Discord như hình dưới

<div align="center">

![image](https://github.com/user-attachments/assets/176f2f1b-6b0f-4d95-bc1e-7f0049bb28d0)

</div>

## Sitemap XML lỗi (Sửa tên sitemap)

Quy trình Workflow

<div align="center">

![image](https://github.com/user-attachments/assets/713bfe31-1946-4bd5-b079-38f34f05ab2d)

</div>

Workflow sẽ thực thi nhánh false tại node quét sitemap và gửi thông báo sau về Discord

<div align="center">

![image](https://github.com/user-attachments/assets/33cb3ae5-d968-481c-ab7c-a89627aed005)

</div>

## Quét site bình thường nếu không lỗi sitemap và domain 

Quy trình Workflow điều hướng

![image](https://github.com/user-attachments/assets/f9e209e8-348f-45b1-bbcc-05a4f2d67322)

Khi này có thể thấy vòng lặp sẽ chạy qua toàn bộ node IF để xét điều kiện index và node IF sẽ duyệt các điều kiện đó để đưa dữ liệu vào Workflow thỏa mãn, Sau mỗi lần hoàn thành Workflow sẽ chạy vào node "NoOp" để chạy 1000 items kế tiếp

Thứ tự Workflow

<div align="center">

![Screenshot from 2025-06-19 16-45-53](https://github.com/user-attachments/assets/d4209e61-6444-472b-b0d6-0aaf519ed469)

</div>

Cách thông báo lỗi hiển thị

<div align="center">

![image](https://github.com/user-attachments/assets/4f23b47f-7df0-4748-83cb-03adf863fc66)

</div>
