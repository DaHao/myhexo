---
title: Creepy Javascript - (0, obj.method)()
tags:
  - CreepyJavascript
index_img: https://images.pexels.com/photos/7035978/pexels-photo-7035978.jpeg
banner_img: https://images.pexels.com/photos/7035978/pexels-photo-7035978.jpeg
categories:
  - - Tech
    - Program
    - Javascript
date: 2022-07-18 09:00:38
---

[Pexels 上由 Peter Lopez 拍攝的相片](https://www.pexels.com/zh-tw/photo/7035978/)

<hr/>

Creepy Javascript 是記錄一些遇到奇怪的 Javascript 的寫法。

寫 Code 寫久了，偶爾會遇到一些看不懂，或者不知道為什麼要這樣寫的東西。

然後查過又忘記，所以就記錄下來，順便分享給大家知道。
<br/>

這次是在 Asyncjs 中看到這樣的寫法。

{% pullquote [class] %}
`var keys = (0, _keys2.default)(tasks);`
{% endpullquote %}
<br/>

雖然知道 `(0, _keys2.default)` 這種寫法會回傳 `_keys2.default`，但是不知道為什麼要這樣寫。

查了一下 Google 找到了這篇 [What is the meaning of this code (0, function) in javascript](https://stackoverflow.com/questions/40967162/what-is-the-meaning-of-this-code-0-function-in-javascript)

這種寫法有幾個特性：

1. 使用 eval 的時候，它可以轉換成 global variable

{% codeblock lang:javascript %}
        (function() {
          // direct call to eval, creates local variable
          eval("var bar = 123");
        })();
        console.log(bar);            // Reference Error

        (function() {
          // indirect call to eval, creates global variable
          (0, eval)("var foo = 123");
        })();
        console.log(foo);            // 123
{% endcodeblock %}

2. 當你想呼叫某個 function，卻不想傳 obj 當 this 的話

{% codeblock lang:javascript %}
        var obj = {
          method: function() { return this; }
        };
        console.log(obj.method() === obj);     // true
        console.log((0, obj.method)() === obj); // false
{% endcodeblock %}

簡單來說，這種 `(0, obj.methed)()` 間接呼叫的方式，保證了該 function 是執行在 global scope 底下。

所以第一個 case 的 foo 會變成全域。

第二個 this 會不等於 obj (因為此時的 this 是 global)。
<br/>

下面這個例子會更加明顯，因此我覺得 `0` 這個數字根本不是重點。

{% codeblock lang:javascript %}
        var x = 'outer';
        (
          function() {
            var x = 'inner';
            eval('console.log("direct call: " + x)');
            (1, eval)('console.log("indirect call: " + x)');
          }
        )();

        // print
        // direct call: inner
        // indirect call: outer
{% endcodeblock %}

