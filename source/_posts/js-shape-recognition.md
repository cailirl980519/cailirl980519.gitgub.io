---
title: 識別HTML5 Canvas上的圖形（直線、箭號、圓形、矩形...）
date: 2021-10-26 10:52:47
categories: Javascript
tags:
- Javascript
- Web
- HTML5 Canvas
- Shape recognition
---

在iOS的備忘錄或Samsung Note App裡畫畫都可以將手畫的筆跡轉換成漂亮的圖形，讓我不禁想嘗試HTML5的Canvas是不是也可以做到，今天就來記錄一下如何將HTML5 Canvas上醜醜的圖形變成漂亮的直線、箭號、矩形等等。

### 開心地展示時間
<p class="codepen" data-height="300" data-slug-hash="zYdNBxM" data-preview="true" data-editable="true" data-user="cailirl980519" style="height: 300px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;">
  <span>See the Pen <a href="https://codepen.io/cailirl980519/pen/zYdNBxM">
  Shape Recognition</a> by cailirl980519 (<a href="https://codepen.io/cailirl980519">@cailirl980519</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://cpwebassets.codepen.io/assets/embed/ei.js"></script>

### 正文

#### HTML
首先，先在頁面創建一個`<canvas>`用來畫畫，再創建一個`<button>`來清除畫布

``` html
<canvas id="canvas"></canvas>
<div id="menu">
  <button id="clear" onclick="reset()">clear</button>
</div>
```

#### CSS
再來用CSS排個版～
``` css
#canvas {
  width: 100vw;
  height: 100vh;
}
#menu {
  position: absolute;
  top: 10px;
  left: 10px;
  z-index: 1;
}
```

#### JS
先宣告幾個等等需要用到的參數，並將偵測動作加入畫布

``` js
const isMobile = window.orientation !== undefined;
const canvas = document.getElementById("canvas");
// 設定畫布大小
canvas.width = canvas.clientWidth;
canvas.height = canvas.clientHeight;
const ctx = canvas.getContext("2d");
// 設定畫布樣式
ctx.fillStyle = "lightgray";
ctx.lineWidth = 2;
ctx.font = "30px Arial";
// 記錄筆跡的參數
var drawCache = new Array();
var lineCache = new Array();
// 偵測畫布動作
if (isMobile) {
  canvas.addEventListener("touchstart", handleDraw);
  canvas.addEventListener("touchmove", handleDraw);
  canvas.addEventListener("touchend", handleDraw);
} else {
  canvas.addEventListener("pointerdown", handleDraw);
  canvas.addEventListener("pointermove", handleDraw);
  canvas.addEventListener("pointerup", handleDraw);
  canvas.addEventListener("pointercancel", handleDraw);
}
```

接下來宣告一個`handleDraw` function來執行偵測到畫布動作後需要做的事情：
- 偵測到這些動作時，將目前觸碰到的點加入`lineCache`陣列裡
    - pointerdown
    - pointermove
    - touchstart
    - touchmove
- 偵測到這些動作時，利用`detectShape`判斷圖形，再將偵測到的圖形及目前的線段`lineCache`新增至`drawCache`
    - pointerup
    - pointercancel
    - touchend

``` js
function handleDraw(e) {
  e.preventDefault();
  if (
    (isMobile && (e.type == "touchstart" || e.type == "touchmove")) ||
    (e.buttons == 1 && (e.type == "pointerdown" || e.type == "pointermove"))
  ) {
    // 將目前觸碰到的點加入`lineCache`陣列裡
    lineCache.push({
      x: isMobile ? e.touches[0].clientX : e.clientX,
      y: isMobile ? e.touches[0].clientY : e.clientY
    });
    draw(); // 更新畫布
  } else if (
    e.type == "pointerup" ||
    e.type == "pointercancel" ||
    e.type == "touchend"
  ) {
    if (lineCache.length > 1) {
      detectShape(lineCache); // 偵測圖形
    }
    lineCache = new Array();
    draw(); // 更新畫布
  }
}
```

接著就是今天的重點`detectShape`
1. 先將三個連續點變為兩個線段，並判斷兩個線段的角度，如果大於`Math.PI / 4`則認定它為角
2. 計算所有認定為角的平均角度
3. 判斷第一個點跟最後一個點的距離是否小於30，如是，則認定為封閉的圖形，並將**第一個點**加入`corners`的最後一項
4. 若為封閉的圖形，則利用`avgA`判斷是否為**圓形**或**橢圓形**
5. 若為封閉的圖形，且不是圓形或橢圓形，並有5個角，則可以認定它為**四方形**，接著就繼續判斷是否為**矩形**
6. 若不為封閉的圖形，則將**最後一個**點加入`corners`的最後一項
7. 若不為封閉的圖形，則判斷是否為曲線、直線或箭號

``` js
function detectShape(points) {
  let ptLast = { ...points[points.length - 1] };
  ptLast.index = points.length - 1;
  // 線段轉彎的地方（角）
  let corners = [points[0]];
  let n = 0;
  let t = 0;
  let lastCorner = points[0];
  let angles = 0;
  let avgA;
  // 1. 判斷是否為角
  for (let i = 1; i < points.length - 2; i++) {
    let pt = points[i + 1];
    let d = delta(lastCorner, points[i - 1]);
    if (len(d) > 20 && n > 2) {
      ac = delta(points[i - 1], pt);
      if (Math.abs(angle_between(ac, d)) > Math.PI / 4) {
        angles += (angle_between(ac, d) * 180) / Math.PI;
        pt.index = i;
        corners.push(pt);
        lastCorner = pt;
        n = 0;
        t = 0;
      }
    }
    n++;
  }
  // 2. 計算所有角的平均角度
  avgA = angles / corners.length;
  // 3. 判斷是否為封閉的圖形
  if (len(delta(points[points.length - 1], points[0])) < 30) {
    // 3. 將第一個點加入corners最後一項
    ptLast = { ...points[0] };
    ptLast.index = 0;
    corners.push(ptLast);
    // 4. 判斷是否為圓形或橢圓形
    if (avgA < 45) {
      let l = points.length;
      let ps = [...points];
      let left = ps.sort((a, b) => a.x - b.x)[0];
      let right = ps.sort((a, b) => b.x - a.x)[0];
      let top = ps.sort((a, b) => a.y - b.y)[0];
      let bottom = ps.sort((a, b) => b.y - a.y)[0];
      corners = [left, right, top, bottom];
      drawCache.push({
        type: "circle",
        corners: corners,
        line: [...lineCache]
      });
      return;
    }
    // 5. 判斷是否為四方形
    if (corners.length == 5) {
      let p0 = corners[0];
      let p1 = corners[1];
      let p2 = corners[2];
      let p3 = corners[3];
      let p0p1 = delta(p0, p1);
      let p1p2 = delta(p1, p2);
      let p2p3 = delta(p2, p3);
      let p3p0 = delta(p3, p0);
      // 5. 判斷是否為矩形
      if (
        Math.abs(angle_between(p0p1, p1p2) - Math.PI / 2) < Math.PI / 6 &&
        Math.abs(angle_between(p1p2, p2p3) - Math.PI / 2) < Math.PI / 6 &&
        Math.abs(angle_between(p2p3, p3p0) - Math.PI / 2) < Math.PI / 6 &&
        Math.abs(angle_between(p3p0, p0p1) - Math.PI / 2) < Math.PI / 6
      ) {
        let p0p2 = delta(p0, p2);
        let p1p3 = delta(p1, p3);
        let diag = (len(p0p2) + len(p1p3)) / 4;
        let tocenter0 = scale(unit(p0p2), -diag);
        let tocenter1 = scale(unit(p1p3), -diag);
        let center = average(corners);
        let angle = angle_between(p0p2, p1p3);
        corners = [
          add(center, tocenter0),
          add(center, tocenter1),
          add(center, scale(tocenter0, -1)),
          add(center, scale(tocenter1, -1)),
          add(center, tocenter0)
        ];
        drawCache.push({
          type: "rect",
          corners: corners,
          line: [...lineCache]
        });
        return;
      }
    }
  } else { // 不是封閉的圖形
    // 6. 將最後一個點加入corners最後一項
    corners.push(ptLast);
    // 7. 判斷是否為曲線
    if (avgA > 20 && avgA < 40) {
      drawCache.push({
        type: "arc",
        corners: corners,
        line: [...lineCache]
      });
      return;
    }
    // 7. 判斷是否為直線
    if (corners.length == 2) {
      drawCache.push({
        type: "line",
        corners: corners,
        line: [...lineCache]
      });
      return;
    } else if (corners.length == 3) { 
      // 7. 判斷是否為鍵號
      let p0 = corners[0];
      let p1 = corners[1];
      let p2 = ptLast;
      let p0p1 = delta(p0, p1);
      let p1p2 = delta(p1, p2);
      if (Math.abs(angle_between(p0p1, p1p2) - Math.PI / 4) > Math.PI / 6) {
        drawCache.push({
          type: "arrow",
          corners: corners,
          line: [...lineCache]
        });
        return;
      }
    }
  }
  drawCache.push({
    type: "other",
    corners: corners,
    line: [...lineCache]
  });
}
```

經過複雜且麻煩的偵測圖形後，最後就只要將偵測到的圖案更新至畫布內就大功告成了！

``` js
// 清除畫布
function clear() {
  ctx.clearRect(0, 0, canvas.width, canvas.height);
}

function draw() {
  clear();
  ctx.strokeStyle = "orange";
  // 繪製完成的筆跡
  drawCache.forEach((e) => {
    ctx.beginPath();
    if (e.type == "line") { // 直線
      let l = e.line.length - 1;
      ctx.moveTo(e.line[0].x, e.line[0].y);
      ctx.lineTo(e.line[l].x, e.line[l].y);
    } else if (e.type == "arrow") { // 箭號
      let l = e.line.length - 1;
      let angle =
        (Math.atan2(e.line[l].y - e.line[0].y, e.line[l].x - e.line[0].x) *
          180) /
        Math.PI;
      let angle1 = ((angle + 150) * Math.PI) / 180;
      let angle2 = ((angle - 150) * Math.PI) / 180;
      ctx.moveTo(e.line[0].x, e.line[0].y);
      ctx.lineTo(e.line[l].x, e.line[l].y);
      ctx.lineTo(
        e.line[l].x + 20 * Math.cos(angle1),
        e.line[l].y + 20 * Math.sin(angle1)
      );
      ctx.moveTo(e.line[l].x, e.line[l].y);
      ctx.lineTo(
        e.line[l].x + 20 * Math.cos(angle2),
        e.line[l].y + 20 * Math.sin(angle2)
      );
    } else if (e.type == "rect") { // 矩形
      ctx.strokeStyle = "orange";
      ctx.beginPath();
      ctx.moveTo(e.corners[0].x, e.corners[0].y);
      e.corners.forEach((p) => ctx.lineTo(p.x, p.y));
    } else if (e.type == "arc") { // 曲線
      ctx.moveTo(e.corners[0].x, e.corners[0].y);
      for (var i = 0; i < e.corners.length - 1; i++) {
        var x_mid = (e.corners[i].x + e.corners[i + 1].x) / 2;
        var cp_x1 = (x_mid + e.corners[i].x) / 2;
        var cp_y1 = i % 2 == 0 ? e.corners[i + 1].y : e.corners[i].y;
        ctx.quadraticCurveTo(
          cp_x1,
          cp_y1,
          e.corners[i + 1].x,
          e.corners[i + 1].y
        );
      }
    } else if (e.type == "circle") { // 圓形或橢圓形
      ctx.ellipse(
        (e.corners[0].x + e.corners[1].x) / 2,
        (e.corners[2].y + e.corners[3].y) / 2,
        Math.abs(e.corners[1].x - e.corners[0].x) / 2,
        Math.abs(e.corners[3].y - e.corners[2].y) / 2,
        0,
        0,
        Math.PI * 2
      );
    } else { // 其他
      ctx.beginPath();
      ctx.moveTo(e.corners[0].x, e.corners[0].y);
      e.corners.forEach((p) => ctx.lineTo(p.x, p.y));
    }
    ctx.stroke();
  });
  // 正在繪製的筆跡
  ctx.strokeStyle = "pink";
  if (lineCache.length > 1) {
    ctx.beginPath();
    ctx.moveTo(lineCache[0].x, lineCache[0].y);
    lineCache.forEach((p) => {
      ctx.lineTo(p.x, p.y);
    });
    ctx.stroke();
  }
  // 沒有任何筆跡
  if (lineCache.length + drawCache.length == 0)
    ctx.fillText(
      "Draw Something Here.",
      canvas.width * 0.4,
      canvas.height * 0.45
    );
}
```