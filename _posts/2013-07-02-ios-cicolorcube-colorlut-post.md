---
layout: post
title: "[iOS] 使用CIColorCube快速製作濾鏡"
description: ""
category: 
tags: []
draft: true
---
{% include JB/setup %}

如果想要再iOS的App內製作照片濾鏡，可以使用Core Image Framework內一系列的CIFilter來實作。CIFilter提供了可以調整亮度、對比等等各種特效的濾鏡。不過如果能直接用Photoshop等影像處理軟體，直接調整這些參數，可以大幅加快開發速度。

本文將介紹透過CIFilter中的CIColorCube來套用Color Lookup Table，快速開發色彩濾鏡。
<!--more-->

###Color Lookup Table
在影像處理的領域中，當我們想要調整一個影像的色彩時，時常會用到Color Lookup Table(簡稱ColorLUT)的技術。

舉個簡單的例子，如果我們想要讓影像中每個像素的R值變成原本的0.3倍，最基本的作法就是把每一個像素的R值乘以0.3，假設影像的大小為1024*768，那麼總共要786432次浮點數除法。

如果我們一開始先建一張表，把所有色彩值經過處理(R值變為0.3倍)之後的結果記錄起來，然後把每個像素的色彩值拿去查表，得到處理之後的色彩值，那麼我們只要做786432查表動作，會比浮點運算快上許多。實際上大部分色彩調整的演算法都比這個例子複雜許多，因此更能凸顯出查表法的高效率。

不過如果要把所有色彩的處理結果都存起來，那這張表勢必會相當肥大。以RGB 24 bits為例，每個像素佔3 byte，而總共會有16777216(256*256*256)種色彩，總共佔48MB(256*256*256*3 bytes)，看來並不小。因此實務上我們並不會記下所有的色彩，而是只記下部分的色彩，其他不在表內的色彩則用內插法取得處理後的結果。

因為每個像素的色彩都是由RGB三種顏色組成，因此我們會以三維陣列的方式來儲存這張表。如果把三維陣列中的每一個像素想像成三度空間中的一個點，而R、G、B分別代表X、Y、Z的座標，則陣列中的所有像素可以構成一個正立方體。以RGB 24bits為例，由於R、G、B的值為0~255，因此正方體的長寬高為255x255x255。當我們要查某個像素經過處理之後的色彩，只要將該像素處理前的RGB值當做X、Y、Z座標，位於那個位置上的像素則為處理後的顏色。   
<img src="/images/ios_cicolorcube_colorlut/cube.jpg"/> 

但是為了節省大小，我們不可能提供所有的點，因此我們會使用內差法來推算。以4x4x4的ColorLUT為例，X、Y、Z三個方向都只有在0、85、170、255時有資料，如果座標不在這些點上，則必須跟週圍的點做內差來得到色彩的近似值，由於是在三維空間內，因此可以透過三線性內差(Trilinear Interpolation)來計算。

###CIColorCube
在iOS App內可以透過Core Image Framework內的CIColorCube實作Color Lookup Table。使用CIColorCube時必須設定：

- **inputCubeDimension:** 也就是正立方體每邊包含幾個點。以4x4x4的ColorLUT為例，則inputCubeDimension=4
- **inputCubeData:** NSData物件，儲存每個點的RGBA資訊。每個點的R、G、B、A值皆為float(0~1.0)，點的儲存順序為：從Z=0的平面開始，從Y=0的列開始，儲存X=0~X=255的點，後面接著Y=1列
，接著再儲存Z=1的平面。以4x4x4的ColorLUT為例，順序為(0,0,0)、(85,0,0)、(170,0,0)、(255,0,0)、(0,85,0)、(85,85,0)...(0,255,255)、(85,255,255)、(170,255,255)、(255,255,255)

那我們怎麼產生inputCubeData內的資料呢？我們可以先將每個點原本對應的顏色先存在一張圖片內，然後透過Photohop之類的影像處理軟體，套用各種色彩特效到這張圖片，之後App讀取調整過後的圖片，根據每個像素的位置，將色彩資訊填進inputCubeData，之後套用CIColorTable的影像就會跟直接透過Photoshop調整出來的色彩有一樣的效果。

這樣講實在有點抽象，我們以4x4x4的ColorLUT為例，經過Photoshop調整色彩前如下：   
<img src="/images/ios_cicolorcube_colorlut/4x4x4ColorLUT_before.jpg"/>   
可以看到這是一張8x8的圖片，分別包含了四個 4x4的正方形，以右上角正方形為例，這代表Z=0的平面，而X軸為R，Y軸為G，左上角第一個像素代表位於(0,0,0)的點，第二個像素代表位於(85,0,0)的點，以此類推，由於這些像素代表的是未處理前的顏色，因此第一個像素的RGB值為0,0,0，第二的像素的RGB值為85,0,0

接著我們將這張圖經過Photoshop降低色彩飽和度，結果如下：   
<img src="/images/ios_cicolorcube_colorlut/4x4x4ColorLUT_after.jpg"/>   
這張圖內的每個像素，代表處理之後，在每個座標點上的色彩。也就是說，如果我們想知道一個RGB值為85,0,0的像素降低飽和度之後的顏色，可從正立方體中位於(85,0,0)的點得知，也就是這張圖中左上角第二個像素的顏色，RGB值為68,16,16。

因此我們只要將第二張圖片中每個像素的RGB值換算成浮點數之後，依序填入NSData中，產生的CIColorCube就可以做出跟Photoshop中一樣的色彩濾鏡。

