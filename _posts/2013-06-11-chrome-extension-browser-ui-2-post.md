---
layout: post
title: "Chrome Extension Browser UI (2)"
description: ""
category: 
tags: []
---
{% include JB/setup %}

在之前的文章已經介紹了Browser Action、Page Actions、Context Menus、Desktop Notifications的用法。

本文將簡單介紹Omnibox、Options Page、Override Page的使用方式。
<!--more-->
以下使用的範例程式可在以下網址取得   
<https://github.com/huangtw/ChromeExtensionExamples/tree/master/browserUI>

###Omnibox
當使用者在網址列輸入特定keyword時，chrome extension可以透過omnibox來讓使用者在網址列輸入指令，並且在輸入的過程中顯示提示，並且在輸入完成之後根據輸入的指令進行處理。

廢話不多說，直接看範例：   
**manifest.json**
{% highlight json %}
{
  "name": "Omnibox Test",
  "version": "1.0",
  "description": "Test Omnibox",
  "omnibox": {
    "keyword" : "test"
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

- **omnibox:** 當使用者在網址列輸入test這個keyword時將會啟用omnibox
- **background:** 在background.js中處理網址列的要出現什麼提示以及如何處理指令
- **permissions:** 由於本範例會根據使用者輸入的內容將現在的分頁連結到google搜尋圖片，因此需要取得呼叫chrome.tabs.* API的權限

**background.js**
{% highlight javascript %}

chrome.omnibox.onInputChanged.addListener(function(text, suggest) {
  words = ["air", "apple", "book", "bus"];
  suggestions = [];

  for (i in words) {
    if (words[i].indexOf(text) == 0) {
      suggestions.push({
        content: words[i],
        description: "<match>" + text + "</match>" + "<dim>" + words[i].replace(text, "") + "</dim>"
      });
    }
  }

  suggest(suggestions);
});

chrome.omnibox.onInputEntered.addListener(function(text) {
  chrome.tabs.getSelected(null, function(tab) {
    chrome.tabs.update(tab.id, {url: "https://www.google.com.tw/images?q=" + text});
  });
});

{% endhighlight %}

我們透過chrome.omnibox.onInputChanged.addListener與chrome.omnibox.onInputEntered.addListener來設定onInputChanged與onInputEntered時的callback。

當使用者輸入的內容改變時（例如多打一個字），會觸發onInputChanged event。當使用者輸入完畢按下enter或者點選提示時，將會觸發onInputEntered。

我們在onInputChanged的callback中，比對使用者輸入的字是否包含在"air"、"apple"、"book"、"bus"這四個字的開頭，如果有則把符合的字加到提示裡。

每個提示包含了content與description，description是顯示在網址下拉選單的提示內容，而當使用者點選這個提示時，則等同於直接在網址列輸入content的內容。

description支援使用XML tag來設定文字外觀。目前支援url（變色）、match（加粗）、dim（變淡）。我們將使用者輸入的字用match tag來加粗強調，而將單字剩下的內容用dim使它變淡，增加識別度。

最後透過suggest這個callback將包含整個提示的array設到網址列。

當使用者輸入完成後，在onInputEntered的callback中，我們將現在分頁的url導向google圖片搜尋並且以輸入的內容為關鍵字，搜尋相關的圖片。

測試一下！   
<img src="/images/chrome_extension_browser_ui_2/omnibox.jpg"/>   
當我們輸入test並且按空白鍵之後，會出現Omnibox Test的字樣，表示已經啟動了Omnibox。接著輸入a，下面會顯示出air與apple，點選之後就會跳往google搜尋相關的圖片！

###Options Page
Option Page提供使用者一個網頁變更一些關於該chrome extension的設定。Options Page的使用方是相當簡單，就跟寫一般網頁差不多。

使用方式如下：   
**manifest.json**
{% highlight json %}
{
  "name": "Options Page Test",
  "version": "1.0",
  "description": "Test Options Page",
  "options_page": "options.html",
  "manifest_version": 2
}
{% endhighlight %}

- **options_page:** 要當作Options Page的HTML檔

**options.html**
{% highlight html %}
<h1>Hello!! This is an options page!!</h1>
{% endhighlight %}

這裡的範例相當簡單，設定頁面只是單純秀一行字，如果真的要儲存或修改設定的話，通常會使用localStorage的API來儲存設定，在此就不多做說明。

測試一下！   
<img src="/images/chrome_extension_browser_ui_2/options_page_1.jpg"/>   
可以看到在擴充功能的頁面內，該Extention的項目下多了"選項"可以使用，如果有使用Page Action或者Browser Action的話，在圖示上點選右鍵也可以看到"選項"

<img src="/images/chrome_extension_browser_ui_2/options_page_2.jpg"/>   
點選之後會開啟設定頁面

至於設定頁面到底該放些什麼，那又是另一個故事了......

###Override Page
Override Page可以把原本Chrome的"書籤管理員"、"紀錄"、"新分頁"其中一頁取代成自己的頁面。使用時只要在manifest.json指定要取代的頁面以及拿來取代的HTML檔即可

使用方式如下：   
**manifest.json**
{% highlight json %}
{
  "name": "Override Page Test",
  "version": "1.0",
  "description": "Test Override Page",
  "chrome_url_overrides" : {
    "newtab": "newTab.html"
  },
  "manifest_version": 2
}
{% endhighlight %}

- **chrome_url_overrides:** 在這裡我們要把新分頁取代成new_tab.html，因此使用"newtab"，如果要取代"書籤管理員"或"紀錄"則使用"bookmarks"或"history"，三種只能擇一。

**newTab.html**
{% highlight html %}
<h1>My New Tab!!!</h1>
{% endhighlight %}

測試一下！   
<img src="/images/chrome_extension_browser_ui_2/override_page.jpg"/>   
新分頁的內容已經被取代掉了！

值得注意的是，如果有多個Chrome Extension都要取代同樣的頁面，則只有最後安裝的那個Extension能取代成功。

關於Chrome Extension Browser UI的簡單介紹就在此告一段落，之後有時間會繼續介紹一些相關應用的實作方式。