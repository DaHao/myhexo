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

> User 如果直接關掉 browser 的話，terminal 的 websocket 連線需要中斷並刪除 pod

困難點在哪？  
---
在 socket-io v3 的版本中，只要一按 browser tab 的 [x] 按鈕，socket 馬上就會 disconnect，不管有沒有 `confirm` 之類的動作都一樣    

因為 socket disconnect，delete pod 的 action 就沒辦法送到 backend 進行處理  

Solution   
---
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

結論
---
如果要升級 socket-io 的話，不只有 xportal 需要升級，wtf 也需要做升級   

因此問題暫不處理，等到升級後再回來看

Ref.
---
https://github.com/socketio/socket.io-client/issues/1451


在 yee 的協助下測出了更多 knowhow

1. 使用 socket-io v2 版本的 socket 連線不會中斷 (目前為 v3)

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
