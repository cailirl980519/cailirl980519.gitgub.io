---
title: 透過Pointer events偵測多點觸控
date: 2021-10-22 14:17:53
categories: Javascript
tags: 
- Javascript
- Web
- Pointer events
---

在多平台的時代，瀏覽器的輸入事件也越來越多樣化，從早期只有滑鼠可以點擊畫面，到現在有了觸控、觸控筆，於是Pointer events誕生了，它不僅保留了Mouse events的常見屬性，還多了可以追蹤各種類型指針事件的功能，使程式碼可以相容各種不同類型的裝置。

Pointer events不像Touch events一樣，Pointer events並沒有TouchesList，沒辦法輕易得知目前有幾的觸碰點在畫面上，所以如果要進行縮放這種需要兩隻手指的動作，就需要多寫點程式碼來判斷了，今天就來記錄一下如何用一個事件只能記錄一個觸碰點的Pointer events來偵測多點觸控～

### 先來個成果時間
<p class="codepen" data-height="300" data-default-tab="result" data-slug-hash="VwzmQXJ" data-preview="true" data-editable="true" data-user="cailirl980519" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;">
  <span>See the Pen <a href="https://codepen.io/cailirl980519/pen/VwzmQXJ">
  pointer-events</a> by cailirl980519 (<a href="https://codepen.io/cailirl980519">@cailirl980519</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://cpwebassets.codepen.io/assets/embed/ei.js"></script>

### 正文

一開始，我們先將需要偵測的物件加入事件偵測

``` js
const t = document.getElementById("target")
t.addEventListener('pointerdown'	, zoomHandler, false)
t.addEventListener('pointermove'	, zoomHandler, false)
t.addEventListener('pointerup'		, zoomHandler, false)
t.addEventListener('pointercancel'	, zoomHandler, false)
t.addEventListener('pointerout'		, zoomHandler, false)
t.addEventListener('pointerleave'	, zoomHandler, false)
```

接下來將每個事件要做的動作加入`zoomHandler`：
- pointerdown: 將觸碰點加入`evCache`
- pointermove: 找到移動的點並更新至`evCache`
- pointerup, pointercancel, pointerout, pointerleave: 將離開的點從`evChache`中刪除

最後判斷畫面上觸碰點的數量，隨著數量的不同，變更目標背景的顏色，用Pointer events偵測多點觸控就這樣完成了～
``` js
var evCache = new Array() // 紀錄正觸碰的點
function zoomHandler(e) {
    e.preventDefault()
    if (e.type == "pointerdown") { // 將觸碰點加入`evCache`
        evCache.push(e)
    } else if (e.type == "pointermove") { // 找到移動的點並更新至`evCache`
        for (var i = 0; i < evCache.length; i++) {
            if (e.pointerId == evCache[i].pointerId) {
                evCache[i] = e
                break
            }
        }
    } else {
        for (var i = 0; i < evCache.length; i++) { // 將離開的點從`evChache`中刪除
            if (evCache[i].pointerId == e.pointerId) {
                evCache.splice(i, 1)
                break
            }
        }
    }
    switch (evCache.length) {
        case 0:
            $(t).css("background-color", "transparent")
            $("#text").html("Touch Here")
            break
        case 1:
            $(t).css("background-color", "lightgray")
            $("#text").html("1 Touches")
            break
        case 2:
            $(t).css("background-color", "lightblue")
            $("#text").html("2 Touches")
            break
        case 3:
            $(t).css("background-color", "pink")
            $("#text").html("3 Touches")
            break
    }
}
```