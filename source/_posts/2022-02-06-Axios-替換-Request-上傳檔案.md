---
title: Axios 替換 Request 上傳檔案
tags:
  - Troubleshooting
index_img: 'https://cdn.pixabay.com/photo/2016/12/30/17/27/cat-1941089_1280.jpg'
banner_img: 'https://cdn.pixabay.com/photo/2016/12/30/17/27/cat-1941089_1280.jpg'
date: 2022-02-06 15:18:30
categories:
  - [Tech, Program]
---

# Axios 替換 Request 上傳檔案
最近因為 nodejs 的 [request library](https://www.npmjs.com/package/request) 在 2020 年 2 月 的時候完全的 deprecated，進入了維護狀態，而且不會再有新的功能出現。
再加上目前的專案混雜了 request 及 [axios](https://www.npmjs.com/package/axios) 兩種功能相近的 library，所以興起了全面用 axios 替換掉 request 的念頭。

本來以為是滿單純的替換，不過還是撞到了一些問題……( ˘･з･)

## 阿伯～初四啦！阿伯！
我們原本有一個功能是使用 request 來進行上傳檔案的動作，原始碼大致如下：
```js
const fs = require('fs');
const request = require('request');

function uploadFile(req) {
  const { headers } = req;
  const filePath = "/location/file.txt"
  const options = {
    constmethod: 'PUT',
    url: 'http://ip:port/api/v2/upload/file/',
    header,
    json: true,
    formData: {
      data: {
        value: fs.createReadStream(filePath),
        options: { filename: 'file.txt' },
      }
    }
  };

  request(options, (err, httpRes, body) => {
    // do something
  });
}
```

在網路上查了一下 axios 怎麼做上傳之後，使用 [form-data](https://www.npmjs.com/package/form-data) 輔助，改寫成如下的程式碼：
```js
const fs = require('fs');
const axios = require('axios');
const FormData = require('form-data');

function uploadfile(req) {
  const { headers } = req;
  const filePath = "/location/file.txt"

  const formData = new FormData();
  formData.append(
    'data',
    fs.createReadStream(filePath),
    { filename: 'file.txt' },
  );

  const options = {
    method: 'PUT',
    url: 'http://ip:port/api/v2/upload/file/',
    headers: formData.getHeaders(),
    data: formData,
  };

  axios.request(options)
    .then((res) => { console.log(res); })
    .catch((err) => { throw err; });
}
```

測試之後，發現 server 會回傳 400 Bad request。
```json
  response: {
    status: 400,
    statusText: 'Bad Request',
    ...
    data: { detail: `40000: Failed to upload file: "'data'"` }
  }
```
## 問題在哪兒？
既然 request 可以上傳的話，那麼問題應該是出在 axios 少了什麼東西才對。 ( • ̀ω•́ )

由錯誤訊息判斷，覺得可能是 form data 的問題，於是開始嘗試了各種改寫，不過結果都差不多。 (〒︿〒)

只好去比對用 request 打 api 跟 axios 打 api 到底有什麼差異。
比對之後發現 axios 少了 content-length (其實不只少 content-length，不過測試後發現這個才是原因)。

可是為什麼 request 會自動幫我們加上 content-length 呢？
如果 axios 不會自動加上這個 header 的話，網路上的範例應該都會註明到這點才對啊？

感覺有些貓膩在裡面。 ಠ_ಠ 

## 真相只有一個！
去追查之後發現 request 與 axios 添加 content-length 的判斷邏輯不一樣。

簡單來說 axios 判斷如果 data 類型不是 stream 的話，才會去加上 content-length header。
```js
    if (data && !utils.isStream(data)) {
      if (Buffer.isBuffer(data)) {
        // Nothing to do...
      } else if (utils.isArrayBuffer(data)) {
        data = new Buffer(new Uint8Array(data));
      } else if (utils.isString(data)) {
        data = new Buffer(data, 'utf-8');
      } else {
        return reject(createError(
          'Data after transformation must be a string, an ArrayBuffer, a Buffer, or a Stream',
          config
        ));
      }

      // Add Content-Length header if data exists
      headers['Content-Length'] = data.length;
    }
```

因為我們一開始改寫的 formData 會被 axios 判定為 stream 類型，自然就沒有 content-length。
```js
const fs = require('fs');
const FormData = require('form-data');

const filePath = './test.txt';
const formData = new FormData();
formData.append(
  'data',
  fs.createReadStream(filePath),
  { filename: 'file.txt' },
);

function isObject(val) {
  return val !== null && typeof val === 'object';
}

function isFunction(val) {
  return toString.call(val) === '[object Function]';
}

function isStream(val) {
  return isObject(val) && isFunction(val.pipe);
}

console.log('formData is stream: ', isStream(formData));
// formData is stream:  true
```

那麼問題來了，request 又是怎麼做 content-length 的判斷呢？

request 如果判斷資料有 formData 但沒有帶 content-length header 的話，會使用 getLength 這個函式來取得 length。
```js
    if (self._form && !self.hasHeader('content-length')) {
      // Before ending the request, we had to compute the length of the whole form, asyncly
      self.setHeader(self._form.getHeaders(), true)
      self._form.getLength(function (err, length) {
        if (!err && !isNaN(length)) {
          self.setHeader('content-length', length)
        }
        end()
      })
    } else {
      end()
    }
```

getLength 剛好就是 form-data 所提供的函式，意外發現 request 內部也是使用 form-data 去處理 FormData 的資料。

到這邊再度改寫 axios 的程式加上 content-length，終於可以成功上傳檔案了！
ヽ( ° ▽°)ノ
```js
const fs = require('fs');
const axios = require('axios');
const FormData = require('form-data');

const getLen = (formData) => {
  return new Promise((resolve, reject) => {
    formData.getLength((err, len) => {
      if (!err && !isNaN(len)) { resolve(len); }
      else { reject(err); }
    });
  });
}

async function uploadSolution(req) {
  const { headers } = req;
  const filePath = "/location/file.txt"
  const formData = new FormData();
  formData.append(
    'data',
    fs.createReadStream(filePath),
    { filename: 'gmn_container.gsp' },
  );
  const contentLen = await getLen(formData);

  const config = {
    baseURL: 'http://10.112.1.3:31215/',
    headers: {
      'x-api-host': 'goc',
      'x-api-key': '129ce429-a861-42b7-9929-a6a88a4dcf04',
    },

    auth: {
      username: 'admin',
      password: 'admin',
    },
  };

  const options = {
    url: 'http://ip:port/api/v2/upload/file/',
    method: 'PUT',
    headers: {
      ...header,
      ...formData.getHeaders(),
      'content-length': contentLen,
    },
    data: formData,
  };

  axios.request(options)
    .then((res) => { console.log(res); })
    .catch((err) => { throw err; });
}
```
<br/>
