### Requirement (需求)
客戶原本將 NAS 當作資料庫，使用者直接打開 NAS 指定目錄將圖檔放進去，因應公司流程改善，要改由網頁系統進行圖檔上傳
```
1. 限定圖片上傳，並且有大量上傳需求 (至多 1000 個圖片檔)

2. 上傳的圖檔為高機密資料，須確保資料不落地原則。因此後端系統必須對接客戶既有圖檔系統，透過該系統提供的 API 介面上傳圖檔。

3. client-end 是 google chrome 瀏覽器
```

### Design (設計規劃)
```
前端 : 
      透過 javascript 迴圈方式將使用者選取欲上傳的圖片檔一個一個透過 ajax 方式發送請求
      
後端 :
      將接收到的圖檔，透過 http client 程式呼叫客戶圖檔系統進行資料傳輸
```

### Issue (議題)
```
客戶 : 
      反應上傳圖片需要等待非常久的時間，三百多個檔案需要約莫八分鐘完成作業
      
後端程式 :
      根據 AP log 紀錄發現兩個問題
      1. 此次議題主要探討 - 後端系統接收到的請求是有規律的間隔，但是前端是迴圈直接透過 ajax 發送
      2. 放大問題的火苗 - 後端程式發現對接客戶圖檔系統，一次完整求請求回覆需要耗時 9 秒
```

### Cause (原因)
```
如上述議題提到，後端系統接收到的請求是有規律的間隔，我們往前先追朔發送請求的端點，前端 javascript 的程式是否寫錯導致 ajax 請求發送有等待現象。

但是根據 log 顯示，ajax 請再在迴圈處理下幾乎是快速將 300 個圖檔送出。

進一步使用 chrome 瀏覽器功能 netlog 功能，從日誌中發現瀏覽器發送請求實有等待 socket pool 資源的情況，

最後根據關鍵字 chrome browser connection pool size 找到官方相關文件提及 「6 sockets per destination host」。

因為後端上傳客戶圖片系統需要耗時 9 秒，導致前端連線池資源會被占據，最後才會出現前端瀏覽器發送請求有出現延遲現象
```
1. netlog 紀錄

![image](https://user-images.githubusercontent.com/30015888/139278433-c8b8fc0c-af01-4bcd-9f08-58d1ff86bd16.png)

2. chromium 文件

![image](https://user-images.githubusercontent.com/30015888/139280754-fd0ef777-1955-41a0-a6bd-7e436e9dba59.png)


3. chromium 開發原則基於 RFC7230 規範

![image](https://user-images.githubusercontent.com/30015888/139280878-f8badd2f-6875-4b22-abfc-658313d39ee4.png)

### conclusion (結語)
```
藉著這次問題更加了解，小至一個上傳圖檔功能的開發因為中間經過太多媒介導致效能議題

此案例就是因為後端系統對接效能低落，放大了瀏覽器程式的干涉！
```

### Reference (參考)
[google chrome netlog 功能](https://www.chromium.org/for-testers/providing-network-details)

[chromium 開發文件 TOC-Connection-Management](https://www.chromium.org/developers/design-documents/network-stack#TOC-Connection-Management)

[IETF RFC7230](https://datatracker.ietf.org/doc/html/rfc7230#section-6.4)

