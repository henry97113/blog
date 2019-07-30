---
title: 使用 IntersectionObserver 達到 lazy loading 效果
date: 2019/7/31 00:00:00
tag: [javascript, performance]
catagories: Programming
---

最近看到[這篇](https://developers.google.com/search/docs/guides/javascript-seo-basics) SEO 相關文章中有提到使用 lazy loading 以降低效能使用（不確定是否對 SEO 有幫助？），才發現沒有研究過 lazy loading 的實作，甚至沒聽過 Intersection Observer。

<!-- more -->

## Lazy loading 的實作方法

在 IntersectionObserver 出現之前，常見的實作方法有以下兩種，目的都是要取得要監聽的元素當前之於 viewport 的位置：

+ 在指定 DOM 元素下監聽 `scroll` 事件
+ 每隔一段固定時間執行 `element.getBoundingClientRect()`

兩種方法都可以達到目的，但缺點是會拖累網頁的效能，因為每次執行 `getBoundingClientRect()` 都會[觸發瀏覽器 re-layout 整個頁面](https://gist.github.com/paulirish/5d52fb081b3570c81e3a)，這也是為什麼 IntersectionObserver 會被設計出來的原因

## 如何使用 IntersectionObserver

IntersectionObserver 其實就是使用 [Observer](https://pawelgrzybek.com/the-observer-pattern-in-javascript-explained/) 的設計模式，當需要監聽時 `observe`，不需要時 `unobserve`

先看範例程式碼

```javascript
const io = new IntersectionObserver(entries => {
  entries.forEach(entry => {
    console.log(entry);
  })
}, {
  /* 非必要選項，不填入會套用預設值 */
});

// 開始監聽元素
io.observe(element);

// 停止監聽元素
// io.unobserve(element);

// 停止 io 監聽全部元素
// io.disconnect()
```

當沒有帶入第二個選填參數時，`io` 的 callback function 會在任一被監聽元素進入/離開 viewport 時被執行，而 callback 的參數 `entries` 是一個由 [`IntersectionObserverEntry`](https://developer.mozilla.org/en-US/docs/Web/API/IntersectionObserverEntry) 所組成的 array，我們接下來就是要對它進行操作

如果需要監聽多個元素，像是頁面中有許多圖片待載入，建議將全部元素使用同一個 IntersectionObserver 監聽即可，例如：

```javascript
const io = new IntersectionObserver(entries => {
 // 對 entries 的操作
 // ......
 // ......
});

io.observe(element1);
io.observe(element2);
io.observe(element3);
io.observe(element4);
io.observe(element5);
// ......
```

## IntersectionObserverEntry 是什麼＆有什麼用處

大家可以使用下面範例玩看看（記得打開 console 看印出的結果）印出的 entries/entry 是什麼（在本頁面或是 codepen 都可以）

<p class="codepen" data-height="400" data-theme-id="dark" data-default-tab="js,result" data-user="chou07" data-slug-hash="KOWYVV" style="height: 400px; box-sizing: border-box; display: flex; align-items: center; justify-content: center; border: 2px solid; margin: 1em 0; padding: 1em;" data-pen-title="Basic IntersectionObserver">
  <span>See the Pen <a href="https://codepen.io/chou07/pen/KOWYVV/">
  Basic IntersectionObserver</a> by Henry (<a href="https://codepen.io/chou07">@chou07</a>)
  on <a href="https://codepen.io">CodePen</a>.</span>
</p>
<script async src="https://static.codepen.io/assets/embed/ei.js"></script>

可以看到印出來的 entry 結構長這樣

```javascript
🔽[IntersectionObserverEntry]
    time: 3893.92
    rootBounds: null
    isIntersecting: false
    isVisible: false
  🔽boundingClientRect: ClientRect
    // ...
  🔽intersectionRect: ClientRect
    // ...
    intersectionRatio: 0.1669006496667862
  🔽target: div#box
    // ...
```

這裡簡單說明每個屬性的意思：

+ `time` 是從開始監聽後到實際觸發 callback 執行中間間隔的時間有多久
+ `rootBounds` 是 root element 執行 `getBoundingClientRect()` 的結果，我們沒有設定 root element，所以是 `null`
+ `boundingClientRect` 是被監聽元素執行`getBoundingClientRect()` 的結果
+ `intersectionRect` 是當 callback 被執行時被監聽元素的可視區域
+ `intersectionRatio` 與 `intersectionRect` 類似，告訴我們目前被監聽元素的可視區百分比（之於被監聽元素的總面積），100% 就表示該元素『全部』都在 root element 的畫面當中
+ `isVisible` 為 `true` 表示被監聽元素 100% 在 root element 可視區內
+ `isIntersecting` 如果為 `true` 表示被監聽元素在 root element 可視區內，但不一定是 100%
+ `target` 則是觸發 callback 的被監聽元素

這裡很重要的一點是 IntersectionObserver 是非同步的監聽 root element / observed element 之間的變化，而 callback 只會在瀏覽器 idle 時才會被觸發執行。如此一來可以大幅降低效能需求

不過 callback 執行是在主執行緒上（main thread），所以如果在 callback 一次執行太多事情還是會造成 blocking

## IntersectionObserver's options

上面有提到當建立 IntersectionObserver instance 時，第二個參數是選填的 options，其中包含了

+ `root`：填入的元素會被當成 root element，如果填入 `null` 就會用 viewport 作為 root
+ `rootMargin`：root element 與被監聽元素之間的距離，預設是 `0px`
+ `thresholds`：可以是 0 或 1 的數字，或是由 0 - 1 之間數字/小數組成的 array，且需由小至大排序；下面單獨說明

當中比較直得注意的是 `thresholds`，可以把它想成百分比，callback 會依照該比例依序被觸發，舉例來說：

+ `thresholds: 0` 就只會在被監聽元素進入/離開 root element 時被觸發
+ `thresholds: [0, 0.25, 0.5, 0.75, 1]` 會使 callback 在被監聽元素進入/離開 root element 0%/25%/50%/75%/100% 時被觸發

而 `root` 和 `rootMargin` 平常應該都不會改動到，除非只想要監聽特定元素會是想要讓 root element 被計算的面積變大/縮小

### 範例

```javascript
new IntersectionObserver(() => {/* ... */}, {
  root: null,
  // 如同 CSS 的 margin，rootMargin 可以是正值或是負值、可以是 px 或是 %
  rootMargin: '0px',
  thresholds: [0, 1]
});
```

## 實作 lazy loading

基於上面學的內容做了個簡單的範例模擬 lazy loading

[範例連結](https://codepen.io/chou07/pen/aeppvB)

滾動到被監聽元素時，才會執行 `fetchImage`、且在兩秒後更新圖片

但其實這個範例並不完美，因為雖然捲動到該元素才會開始更新圖片，但往上捲動時同樣會觸發更新圖片（因為 `thresholds: 0`，所以進入或是離開 root element 都會觸發），只不過我使用同一張圖片所以看不出差異，這部分就交給你優化囉！

## 支援度

從 [caniuse](https://caniuse.com/#feat=intersectionobserver) 可以看出其實目前 IntersectionObserver 的支援度並不是很好，尤其 IE 完全不支援，所以在使用上要注意一下。

## 為什麼這裡的 console 可以印出 codepen 範例的 log

最後，不知道你會不會有疑問，上面 codepen 的範例不是在 iframe 裡面嗎？為什麼在部落格的 console 也印得出結果？

這就是 IntersectionObserver 實用的地方了，即便你的網頁是被嵌在其他網頁當中的，你同樣可以知道使用者什麼時候看到你的網頁

## Reference

+ [Make sure Google can see lazy-loaded content](https://developers.google.com/search/docs/guides/lazy-loading)
+ [IntersectionObserver’s Coming into View](https://developers.google.com/web/updates/2016/04/intersectionobserver)
+ [Intersection Observer](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer)
+ [MDN Intersection Observer 範例](https://developer.mozilla.org/en-US/docs/Web/API/Intersection_Observer_API/Timing_element_visibility)
