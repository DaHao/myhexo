---
title: Socket-io Client disconnected
tags:
  - Troubleshooting
index_img: https://images.pexels.com/photos/13612923/pexels-photo-13612923.jpeg?auto=compress&cs=tinysrgb&dpr=2
banner_img: https://images.pexels.com/photos/13612923/pexels-photo-13612923.jpeg?auto=compress&cs=tinysrgb&dpr=2
categories:
  - Tech
  - Program
  - Javascript
---
[Pexels 上由 Frans van Heerden 拍攝的相片](https://www.pexels.com/zh-tw/photo/13612923/)

<hr/>

# 前言
遇到一個需求，當 User 關掉 Browser 的話，Frontend 需要送 Request 到 Backend 去執行某些動作

# 估狗大神救救我啊～
拜了一下狗狗，發覺好像不是很難，Close Browser 有個叫 `beforeunload` 的 Event 可以觸發

參考了一下資料，產出如下的程式碼

```javascript
  // close window
  useEffect(() => {
    const handleWindowClosed = (event) => {
      if (name) { // do something }
    };

    window.addEventListener('beforeunload', handleWindowClosed);
    return () => {
      window.removeEventListener('beforeunload', handleWindowClosed);
    };
  }, [name]);
```

馬上來測試看看！

# Socket-io client: 下班啦～下班啦～
一測試就馬上翻了第一次車，測試的時候發現，為什麼 Backend 沒有做我送過去的 Request？

再做了多次的測試之後，發現當我一關掉 Browser Tab 的時候，Socket-io 就會馬上就斷開連線(facemood)

因為 `do something` 是靠 Socket-io 連線將 Request 送到後端的，所以一但斷線的話後端就收不到 Request

咦？怎麼會這樣？

好吧，既然你這麼急著走的話，那我加一個 `Confirm` 去中斷你的動作總可以了吧？

哼哼，沒錯！，測試起來還是我太天真了…… Socket-io 還是斷線給我看
<br/>

隔壁的同事好心幫我測試了一下，在他們的 Project 沒有測到斷線的情形。

奇怪了…… 怎麼會這樣呢？(facemood)
<br/>

找了一些資料發現，在 Socket-io v3 的版本中，只要一按 Browser Tab 的 [x] 按鈕，Socket-io 馬上就會 disconnect，不管有沒有做 `confirm` 之類的動作都一樣

隔壁同事是 v2 的版本所以安全下莊……

哪泥？還有這種事？升級後反而不能連線了！

# 所以我說，那個 Solution 呢？




這個功能其實是有辦法達成的
```js
  // socket-io client connect
  const socket = io.connect(
    `${NODE_BACKEND}:${NODE_BACKEND_PORT}`,
    {
      closeOnBeforeunload: false,
      // ... 略
    },
  );

  // close window
  useEffect(() => {
    const handleWindowClosed = (event) => {
      event.preventDefault();
      event.stopImmediatePropagation();
      if (podName && platform.name) {
        // delete pod
      }
      event.returnValue = '';
    };

    window.addEventListener('beforeunload', handleWindowClosed);
    return () => {
      window.removeEventListener('beforeunload', handleWindowClosed);
    };
  }, [podName, platform.name]);
```

遺憾的是 `closeOnBeforeunload` 這個參數只有在 `socket.io-client@4.1.0` 以上才有支援  
當前的 socket.io-client 版本為 3.1.1  

**如果不加 closeOnBeforeunload 硬寫會發生什麼事？**
---

socket.io-client 在使用者關掉 browser tab 的時候，會送出 disconnet 的 request 給 socket server   

因此 socket client 就會直接斷線，也就沒有辦法發任何的 action 到後端  

後端沒收到 action 的情況下，自然不會去 delete pod

Ref.
---
https://github.com/socketio/socket.io-client/issues/1451


在 yee 的協助下測出了更多 knowhow


2. 在特定情形下使用 `event.stopImmediatePropagation()` 可以保持連線不中斷

  **使用 class component 且在 constructor 中註冊 beforeunload 事件**
  function component 不管有沒有用 useEffect 連線都會中斷
  比較正確的說法是要在 socket create 前註冊 [link](https://github.com/socketio/engine.io-client/issues/658#issuecomment-801506732)
```js
class TerminalClass extends Component {
  constructor(props) {
    super(props);

    window.addEventListener('beforeunload', (event) => {
      event.preventDefault();
      event.stopImmediatePropagation();
      event.returnValue = '';
    });
  }

  render() {
    // ...略
  }
}
```

3. 較上層的 window 事件會覆蓋掉下層的事件
```js
class App extends Component {
  constructor(props) {
    super(props);

    window.addEventListener('beforeunload', (event) => {
      console.log('hello app');
      event.preventDefault();
      event.stopImmediatePropagation();
      event.returnValue = '';
    });
  }

  render() {
    return <TerminalClass ... />;
  }
}

class TerminalClass extends Component {
  constructor(props) {
    super(props);

    window.addEventListener('beforeunload', (event) => {
      console.log('hello terminal');
      event.preventDefault();
      event.stopImmediatePropagation();
      event.returnValue = '';
    });
  }

  render() {
    // ... 略
  }
}

// 執行後只會出現 'hello app'
```
