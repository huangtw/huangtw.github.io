---
layout: post
title: "[Chrome Extension] Browser UI (1)"
description: ""
category: 
tags: []
---
{% include JB/setup %}

Chrome Extension除了可以背景執行javascript外，最重要的是它可以變成Chrome瀏覽器的一部分，這樣才算一個名符其實的"擴充功能"，因此chrome提供了幾個API讓extension可以把自己融入到Chrome的browser UI中。

本文將簡單介紹Browser Action、Page Actions、Context Menus、Desktop Notifications的用法。
<!--more-->
以下使用的範例程式可在以下網址取得   
<https://github.com/huangtw/ChromeExtensionExamples/tree/master/browserUI>

###Browser Actions
Browser Action是一個很常見的Browser UI，它可以在Chrome網址列的右邊增加一個按鈕。很多Chrome Extension都透過Browser Action來與使用者互動。  

必須注意的是，如果按鈕的功能只適用於特定的網站，則不適合使用Browser Action，這種情況用Page Action比較好，下面會提到！

要加入Browser Action非常簡單，只要在manifest.json中宣告即可，如下：

{% highlight json %}
{
  "name": "Browser Action Test",
  "version": "1.0",
  "description": "Test Browser Action",
  "browser_action": {
    "default_icon": "icon.png",
    "default_title": "Browser Action Test"
  },
  "manifest_version": 2
}
{% endhighlight %}

其他欄位之前已經解釋過，我們把重點放在**"browser_action"**，大致的意義如下：

- **default_icon:** 按鈕圖示
- **default_title:** 當滑鼠停在按鈕上時顯示的提示

測試一下！果然已經多出了一個按鈕，輕輕鬆鬆！   
<img src="/images/chrome_extension_browser_ui_1/browser_action.jpg"/>

###Page Actions
相較於Browser Action是適用於每一個網頁，Page Action則是針對特定的網站或者特定內容的網頁。   

必須注意的是，一個chrome extension只能選用其中一種，不能同時使用Browser Action與Page Action！

Page Action使用方式與Browse Action類似，不過略有不同，如下：

**manifest.json**
{% highlight json %}
{
  "name": "Page Action Test",
  "version": "1.0",
  "description": "Test Page Action",
  "page_action": {
    "default_icon": "icon.png",
    "default_title": "Page Action Test"
  },
  "background": {
    "scripts": [
      "background.js"
    ]
  },
  "permissions": [
    "tabs"
  ],
  "manifest_version": 2
}
{% endhighlight %}

- **page_action:** 這個欄位內的內容與Browser Action一樣，就不再贅述
- **background:** 由於Page Action預設是隱藏的，必須透過javascript將它show出來
- **permissions:** 由於我們在background.js內會檢查tab的URL來決定是否顯示Page Action，因此必須取得使用chrome.tabs.*這些API的使用權限

**background.js**
{% highlight javascript %}

function checkForValidUrl(tabId, changeInfo, tab) {
  if (tab.url.indexOf("www.google.com") > -1) {
    chrome.pageAction.show(tabId);
  }
};

chrome.tabs.onUpdated.addListener(checkForValidUrl);
{% endhighlight %}

在background.js中，我們透過chrome.tabs.onUpdated.addListener來監控tab的狀態，每當tab連結到別的URL或者重新整理時，就會產生update event，此時會呼叫checkForValidUrl這個callback。   

在checkForValidUrl中我們檢查網址是否包含"www.google.com"，如果包含的話就顯示Page Action。

當tab連結到一個URL或者重新整理時，Page Action預設是隱藏的，當我們呼叫了show才會出現在tab的網址列右邊。此時如果我們連結到別的URL，Page Action會再次隱藏。

測試一下！只有在google才會顯示Page Action，在youtube則隱藏   
<img src="/images/chrome_extension_browser_ui_1/page_action_show.jpg"/>   
<img src="/images/chrome_extension_browser_ui_1/page_action_hide.jpg"/>

###Context Menus
Context Menu讓你可以在網頁內點選右鍵出現的選單內，加入屬於這個chrome extension的項目。

你可以根據"在什麼東西上按右鍵"來決定要顯示的Context Menu，例如當你在圖片上按右鍵，與在超連結上按右鍵時顯示的Context Menu可以是不同的。

當一個chrome extension有超過一個以上的Contexnt Menu時，會集合成一個項目，透過彈出子選單的方式來顯示出所有的項目。

用法如下：

**manifest.json**
{% highlight json %}
{
  "name": "Context Menu Test",
  "version": "1.0",
  "description": "Test Context Menu",
  "background": {
    "scripts": [
      "background.js"
    ]
  },
  "permissions": [
    "contextMenus"
  ],
  "manifest_version": 2
}
{% endhighlight %}

- **background:** 我們必須透過background.js來產生Context Menu
- **permissions:** 因為我們在background.js中透過chrome.contextMenus.create來產生Context Menu，因此必須取得使用chrome.contextMenus.*這些API的權限

**background.js**
{% highlight javascript %}

function pageOnClick(info, tab) {
  alert("Page is clicked!!");
}

function linkOnClick(info, tab) {
  alert("Link is clicked!!");
}

function imageOnClick(info, tab) {
  alert("Image is clicked!!");
}

function pageLinkOnClick(info, tab) {
  alert("Page or Link is clicked!!");
}

chrome.contextMenus.create({"title": "Page", "contexts":["page"], "onclick": pageOnClick});
chrome.contextMenus.create({"title": "Link", "contexts":["link"], "onclick": linkOnClick});
chrome.contextMenus.create({"title": "Image", "contexts":["image"], "onclick": imageOnClick});
chrome.contextMenus.create({"title": "Page or Link", "contexts":["page", "link"], "onclick": pageLinkOnClick});

{% endhighlight %}

在background.js中我們透過chrome.contextMenus.create來產生四個Context Menu。

- **title:** 就是右鍵選單內項目顯示的文字
- **conexts:** 在哪些地方點擊右鍵時會出現這個項目如果沒填則預設為"page"
- **onClick:** 點擊時的callback

我們分別針對"page"、"link"、"image"個別增加三個context menu項目。因此在網頁內任意處（單純只有文字或空白處）點擊右鍵，可以看到"
Page"的項目，而"Link"跟"Image"則分別會在右鍵點擊超連結與圖片時出現。

此外我們還增加了一個"Page or Link"項目，由於他的contexts包含"page"與"link"，因此在網頁任意處或者超連結上按右鍵都會出現

測試一下！   
<img src="/images/chrome_extension_browser_ui_1/context_menu_page.jpg"/>   
<img src="/images/chrome_extension_browser_ui_1/context_menu_link.jpg"/>   
Page與Link分別顯是在空白處與超連結的右鍵選單，而Page and Link則都會出現。由於項目有多個，因此會以子選單的方式來呈現

<img src="/images/chrome_extension_browser_ui_1/context_menu_image.jpg"/>   
在圖片上按右鍵只會有Image，由於只有一項，因此不會有子選單

###Desktop Notifications
Desktop Notification可以在角落秀出一個通知方塊，而顯示的位置因作業系統而異，Windows會出現在右下角，而Max OSX則是出現在右上角。

使用方式如下：

**manifest.json**
{% highlight json %}
{
  "name": "Desktop Notification Test",
  "version": "1.0",
  "description": "Test Desktop Notification",
  "web_accessible_resources": [
    "notification.png"
  ],
  "background": {
    "scripts": [
      "background.js"
    ]
  },
  "permissions": [
    "notifications"
  ],
  "manifest_version": 2
}
{% endhighlight %}

- **web_accessible_resources:** Notification用到的圖示必須要在這裡宣告才可以使用
- **background:** 透過background.js來顯示與移除notification
- **permissions:** 因為我們在background.js中透過webkitNotifications.createNotification來產生Notification，因此必須取得使用webkitNotifications.*這些API的權限

**background.js**
{% highlight javascript %}
var notification = webkitNotifications.createNotification(
  'notification.png',
  'Test',
  'Desktop Notification'
);

notification.show();
setTimeout(function (){
  notification.cancel()
}, 2000);
{% endhighlight %}

在background.js中，一開始先透過webkitNotifications.createNotification產生一個notification，三個參數分別是：

- **icon url** 這個notification使用的圖示，這裡我們使用json檔內宣告過的notification.png
- **title** notification的標題
- **text** notification的文字內容

接著我們使用notification.show()讓notification顯示出來，一旦notification顯示出來，它並不會自動消失，必須要使用者手動關閉它。因此我們透過setTimeout，在2秒之後呼叫notification.cancel()讓notification消失。

值得注意的是，一個notification canceal之後，它的生命周期就結束了，再呼叫show也不會出現。

測試一下！   
<img src="/images/chrome_extension_browser_ui_1/desktop_notification.jpg"/>    

下次將會繼續介紹Omnibox、Options Pages、Override Pages的使用方式！