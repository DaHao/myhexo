---
title: Socket-io Client disconnected
tags:
  - Troubleshooting
index_img: >-
  https://images.pexels.com/photos/13612923/pexels-photo-13612923.jpeg?auto=compress&cs=tinysrgb&dpr=2
banner_img: >-
  https://images.pexels.com/photos/13612923/pexels-photo-13612923.jpeg?auto=compress&cs=tinysrgb&dpr=2
excerpt: 遇到一個需求，當 User 關掉 Browser 的話，Frontend 需要送 Request 到 Backend 去執行某些動作
categories:
  - Tech
  - Program
  - Javascript
date: 2023-02-22 23:27:33
---

[Pexels 上由 Frans van Heerden 拍攝的相片](https://www.pexels.com/zh-tw/photo/13612923/)
# 前言
遇到一個需求，當 User 關掉 Browser 的話，Frontend 需要送 Request 到 Backend 去執行某些動作

# 估狗大神救救我啊～
拜了一下狗狗，發覺好像不是很難，Close Browser 有個叫 `beforeunload` 的 Event 可以觸發

參考了一下資料，產出如下的 React 程式碼

```javascript
  // close window
  useEffect(() => {
    const handleWindowClosed = (event) => {
      if (name) {
        // do something
      }
    };

    window.addEventListener('beforeunload', handleWindowClosed);
    return () => {
      window.removeEventListener('beforeunload', handleWindowClosed);
    };
  }, [name]);
```

馬上來測試看看！
# Socket-io client: 下班啦～下班啦～
一測試就翻車，跑的時候發現，Backend 不會執行我送過去的 Request

WHY？
<br/>

又做了多次的測試之後，發現當我一關掉 Browser Tab 的時候，Socket-io 就會馬上就斷開連線 ಠ_ಠ

因為 `do something` 是靠 Socket-io 連線將 Request 送到後端的，所以一但斷線的話後端就收不到 Request

咦？怎麼會這樣？
<br/>

好吧，既然你這麼急著走的話，那我加一個 `Confirm` 去中斷你的動作總可以了吧？

測試的結果是不行，Socket-io 還是會斷線給我看
<br/>

隔壁的同事好心幫我測試了一下，在他們的 Project 沒有測到斷線的情形。

奇怪了…… 怎麼會這樣呢？
<br/>

找了一些資料發現，在 Socket-io v3 的版本中，只要一按 Browser Tab 的 [x] 按鈕，Socket-io 馬上就會 disconnect，不管有沒有做 `confirm` 之類的動作都一樣

隔壁同事是 v2 的版本所以安全下莊……

哪泥？還有這種事？升級後反而不能連線了！

# 所以我說，那個 Solution 呢？
在 Sokcet-io 的 Issue list 中，找到了一個 [issue](https://github.com/socketio/socket.io-client/issues/1451) 問了跟我一樣的問題

這個功能其實是有辦法達成的，在 client 端連線的時候多加一個參數 `closeOnBeforeunload`，觸發 `beforeunload` 的時候就不會馬上斷線  

```js
  // socket-io client connect
  const socket = io.connect(
    socketServer,
    {
      closeOnBeforeunload: false,
      // ... 略
    },
  );
```

不過遺憾的是 `closeOnBeforeunload` 這個參數只有在 `socket.io-client@4.1.0` 以上才有支援  

我們當前的 Project 使用的 socket.io-client 版本為 3.1.1  

又因為與其它 component 連動的關係，沒辦法輕易升級 

怎麼辦？要放棄了嗎？

# Another Solution
幸好，issue 中還有提到另一個解法

在 evnet 中加入 `event.stopImmediatePropagation()`，也可以確保連線不中斷！
```javascript
  useEffect(() => {
    const handleWindowClosed = (event) => {
      event.preventDefault();
      event.stopImmediatePropagation();
      if (name) {
        // do something
      }
      event.returnValue = '';
    };

    window.addEventListener('beforeunload', handleWindowClosed);
    return () => {
      window.removeEventListener('beforeunload', handleWindowClosed);
    };
  }, []);
```

不過一開始測試下來，發現也是沒有作用

本來以為被唬弄了，不過又經過一連串測試後，發現這個寫法只有在特定的情境下才能產生作用

`event.stopImmediatePropagation` 只有在
**使用 class component 且在 constructor 中註冊 beforeunload 事件** 
的條件下才有用！

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

在 function component 中不管有沒有使用 useEffect 連線都會中斷

後來在 issue 找到相關說法，作者說要在socket creation 前註冊 [link](https://github.com/socketio/engine.io-client/issues/658#issuecomment-801506732)

不過說老實話，不知道為什麼在 constructor 中才有作用 

因為照理說，我這個 component 應該也是在 socket creation 之後才建立的，不知道為什麼可以產生作用

反正測試起來的結果就是這樣 (´_ゝ`)

<br/>

另外也要注意，class component 中較上層的 window 事件會覆蓋掉下層的事件
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

# 結論
依 React 的趨勢來說，不能用 function component 來解決實在是有點麻煩

最後還是決定等升級，在此做個研究心得記錄 XD
