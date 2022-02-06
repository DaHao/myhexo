---
title: Javascript - 為什麼 Error 是空物件？
date: 2021-07-17 07:02:26
excerpt: 在研究 Async.js waterfall 的時候，感覺像被 javascript 給陰到了一樣
index_img: /img/post/volodymyr-hryshchenko-H27jNHovB-4-unsplash.jpg
banner_img: /img/post/volodymyr-hryshchenko-H27jNHovB-4-unsplash.jpg
tags:
- Troubleshooting
---
# Javascript - 為什麼 Error 是空物件？
在研究 Async.js waterfall 的時候，感覺像被 javascript 給陰到了一樣

考慮下列的程式碼

```js
const Async = require('async');
const axios = require('axios');

async function func1() {
  return axios({
    url: 'http://www.google.com',
    method: 'GET',
  })
    .then(res => {
      console.log('function 1:', res.status, res.statusText);
      return res.status;
    })
    .catch(error => {
      console.log('阿伯～出事了！阿伯！', error);
      throw error;
    });
}

function func2(status, callback) {
  console.log('function 2:', status);
  callback(null, status);
}

Async.waterfall([
  func1,
  func2,
], (error, result) => {
	if (error) console.log(error)
	else {
    console.log('result: ', result);
	}
});
```

沒發生錯誤時會輸出
```
function 1: 200 OK
function 2: 200
result:  200
```

但只要丟出一個錯誤
```js
async function func1() {
    // ...
    .then(res => {
      console.log('function 1:', res.status, res.statusText);
	  throw "Test error";
      return res.status;
    })
    // ...
}
```

碰！

```
function 1: 200 OK
阿伯～出事了！阿伯！ Test error
Error: Test error
    at /Users/hao/Projects/nchc/xportal/backend/node_modules/async/dist/async.js:173:65
    at processTicksAndRejections (internal/process/task_queues.js:93:5)
```

asyncjs 會先拋出一個奇怪的錯誤，如果先去 google 的話，會覺得這個錯誤好像是 unhandledRejection 的問題

所以我開始找程式碼裡到底有哪裡沒加到，然後全部加過一輪後，一樣爆炸

證明這個思路不對，我開始嚐試走別條路

因為錯誤是 waterfall print error 的時候出現

我一度以為是 aysnc function 跟 非async function 不能這樣混用，又測試了多種寫法

結果也通通爆炸，呀哈哈哈～   ( σ ﾟ∀ ﾟ) ﾟ∀ﾟ)σ 
<br/>

最後試著用別的方式印出 error
```js
Async.waterfall([
  func1,
  func2,
], (error, result) => {
	if (error) console.log(JSON.stringify(error))
	else {
    console.log('result: ', result);
	}
});
```
喔喔！事情終於有了進展，錯誤沒有跑出來了！
```
function 1: 200 OK
阿伯～出事了！阿伯！ Test error
{}
```
可是為什麼是空物件呢…？
又經歷了各種嚐試，之後突然想到一件事情
我一直以為是 asyncjs 的寫法錯誤，才會造成 throw 出來的 error 接不到
可是如果是 javascript 的 error 本來就印不出來呢？

從這個思路終於找到問題，不是 asyncjs 不能 async 跟 非async 混用，而是 **你沒辦法用 JSON.stringify() 去顯示 Javascript 的 Error 型態**

[Why are my JS promise catch error objects empty?](https://stackoverflow.com/questions/38513493/why-are-my-js-promise-catch-error-objects-empty)
[Is it not possible to stringify an Error using JSON.stringify?](https://stackoverflow.com/questions/18391212/is-it-not-possible-to-stringify-an-error-using-json-stringify)

主要的原因在於 asyncjs 的 waterfall 在接到 throw error 的時候，會把它包裝成 error 的型態，在 callback 的時候當然沒辦法直接印出來

比較簡單解決的方法是去存取 error 的 message
```js

Async.waterfall([
  func1,
  func2,
], (error, result) => {
	if (error) console.log(error.message)
	else {
    console.log('result: ', result);
	}
});
```
```
function 1: 200 OK
阿伯～出事了！阿伯！ Test error
Test error
```

為了這種初學者的問題卡了半天，學藝不精，真是慚愧 ( ´•̥̥̥ω•̥̥̥` )
