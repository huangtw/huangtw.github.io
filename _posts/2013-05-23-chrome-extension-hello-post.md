---
layout: post
title: "Chrome Extension 開發初體驗"
description: ""
category: 
tags: []
---
{% include JB/setup %}

最近心血來潮想寫個Chrome Extension來玩玩，小小研究了一下。

本文將會介紹Chrome Extension的基本架構以及開發流程。

<!--more-->
以下使用的範例程式可在以下網址取得   
<https://github.com/huangtw/ChromeExtensionExamples/tree/master/hello>

跟開發網頁一樣。Chrome Extension是由HTML,CSS,Javascript來開發。所有extension需要用到的檔案都會放在同一個資料夾底下。

內容大致如下：

- **manifest.json :** 宣告extension名稱、版本、所需權限、用到的檔案...等等
- **backgroung.js :** 檔名並不限定，必須宣告在manifest.json內，啟動extension時會從這些js開始執行（沒錯是"這些"，因為可以有好幾個檔，下面會解釋！）
- **其他:** html,css,js,圖片檔...等等

###manifest.json
首先，我們必須先在manifest.json內宣告一些必要的資訊

{% highlight json %}
{
  "name": "Hello World",
  "version": "1.0",
  "description": "Testing...",
  "background": {
    "scripts": [
    	"helloA.js",
    	"helloB.js",
    	"helloC.js"
    ]
  },
  "manifest_version": 2
}
{% endhighlight %}

大致的意義如下：

- **name:** 顧名思義就是extension將會顯示的名稱
- **version:** 版本
- **background:** 這個欄位內包含的檔案會在extension一啟動時就被載入，並且在背景執行。因為本例子中只想先載入jsvascript，因此使用"scripts"，就像先前提到的，可載入多個javascript，因此檔名必須放在陣列裡（包在[]內，即使只有一個檔也要！），檔案的順序就是載入的順序，也就是程式碼執行的順序
- **manifest_version:** 本manifest格式的版本

###Background Scripts
接下來是javascript的內容，三個js檔的內容如下

**helloA.js**
{% highlight javascript %}
console.log("Hello World");
{% endhighlight %}
**helloB.js**
{% highlight javascript %}
console.log("Hello Word!!");
{% endhighlight %}
**helloC.js**
{% highlight javascript %}
console.log("Hello Excel!?");
{% endhighlight %}

此時做為一個Chrome Extension的基本元件都有了，可以來進行簡單的測試了！

###載入Extension
首先，必須將這個Extension載入chrome。

1. **設定>擴充功能** (或者直接在網址列輸入**chrome://extensions/**)  
2. 勾選 **開發人員模式**
3. **載入未封裝擴充功能**  
<img src="/images/chrome_extension_hello/load.jpg"/>
4. **載入檔案所在的資料夾**
   
###觀看執行結果
此時可以看我們的Extension已經出現在擴充功能的清單中。不過要去哪裡看執行的結果呢？

- 點選 **查看檢視模式** 旁邊的 **_generated_background_page.html**
<img src="/images/chrome_extension_hello/generated_background.jpg"/>
- 在 **Console** 中可以看到執行的結果
<img src="/images/chrome_extension_hello/console.jpg"/>

從console的log訊息可以看出，三個javascript的執行順序分別為 **helloA.js** > **helloB.js** > **helloC.js**  
可見執行的順序跟manifest中宣告的順序一致，載入多個background script時必須特別注意

耶～學到了這裡，已經可以在你的履歷裡加入**Chrome Exetension Development**這一項了！！   
別傻了！這個連UI都沒有的程式連精美的垃圾都稱不上！因此，下次將會介紹瀏覽器有關的UI，敬請期待
