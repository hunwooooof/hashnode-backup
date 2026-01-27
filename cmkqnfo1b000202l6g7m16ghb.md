---
title: "從模板到畫面，Vue 的渲染機制是什麼？（上）"
seoTitle: "Vue 工廠 ── 從模板到畫面，Vue 的渲染機制是什麼？（上）"
seoDescription: "簡單探討 Vue 從模板到畫面的渲染機制，了解虛擬 DOM、vnode 以及編譯和掛載過程。"
datePublished: Fri Jan 23 2026 08:58:59 GMT+0000 (Coordinated Universal Time)
cuid: cmkqnfo1b000202l6g7m16ghb
slug: vue-template-to-dom-1
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1769159217884/2d7ac497-95e2-463b-a200-d4726cc08f95.png
tags: vuejs, frontend-development

---

## 前言

從接觸 Vue 3（以下稱 Vue）到現在已經有快 2 年的時間，卻一直沒有認真啃過文件，了解 Vue 背後到底用了什麼黑魔法，讓原本寫 React 時用來避免不必要 rerender 的 function 概念，在 Vue 的世界裡消失殆盡。

也因為寫了快 2 年，卻還是不夠清楚 Vue 的生命週期，以及一直用到的 onMounted 到底是在 mount 什麼東西，所以誕生了這篇文章，希望寫完後，能幫助自己更了解 Vue 從開發環境走到瀏覽器的過程。

篇幅有點長，所以分成上下兩篇，這篇主要介紹 Virtual DOM、Compile、Mount，也就是網頁初次載入時會遇見的夥伴們。而 Vue 在 Patch（更新畫面）時到底偷偷摸摸做了多少我們不知道的事情（Vue：人家光明正大😠）就留到下篇介紹！若有任何筆誤的地方，敬請不吝指正我！！！🥸

## 你必須先知道的 virtual DOM 以及 vnode

Vue 的渲染機制會反覆出現 virtual DOM 以及 vnode 這兩個名詞和概念，或許你聽過，或許沒有，但如果能更了解他們、閱讀到他們時能在腦海產生畫面，對於理解 Vue 的渲染機制會有很大的幫助。（筆者自己看到 virtual DOM 一詞時，受到 virtual 這個字的影響，一直覺得是個很色彩繽紛又虛無縹緲的東西。深入了解才知道，原來是那麼剛硬生冷又真實存在的 JavaScript Object 🤡）

### Virtual DOM

Virtual DOM 把所有 UI 透過數據結構「虛擬」地表現出來並保存在 JavaScript 的內存中，是由多個 vnode 物件組成的結構。

### vnode（virtual node）

我們舉一個實際的例子，用一段 HTML 來示範，如果變成 vnode 會長怎樣？

```xml
<div id="greetings">Hello!</div>
```

這段 HTML 經過 Vue 的處理後，輸出的 vnode 會長這樣，一個 JavaScript Object！

```javascript
const vnode = {
  type: 'div',
  props: {
    id: 'greetings'
  },
  children: 'Hello!'
  // ...其他 key
}
```

> 實際的 vnode 會有更多欄位，這些欄位提供效能上的幫助，減少渲染時不必要的處理。
> 
> Vue 生成 vnode 的方式不止一種，其中包含[渲染函数 API](https://cn.vuejs.org/api/render-function.html)。

## 從模板開始，一路經過編譯（Compile），掛載（Mount）到畫面上

從 template 到出現在瀏覽器畫面上的中間這段，到底發生了什麼事呢？Virtual DOM 又是在哪裡出來的呢？

假設今天我們寫了這段，並且打包起來：

```xml
<div>
  <div>Animals</div>
  <div :class="{ active }">
    <p>dog</p>
  </div>
  <div>{{ value }}</div>
</div>
```

### Compile 編譯

在打包的過程中，Vue 會將 template **編譯**成渲染函數（Render functions，會回傳 vnode 的函數）。

動態資料的部分，Vue 也會在瀏覽器 runtime compiler 做即時編譯。

```javascript
import { ... } from 'vue'

export function render(_ctx, _cache) { // A render function
  return (
    openBlock(),
    createElementBlock('div', null, [
      createElementVNode('div', null, 'Animals'), // <div>Animals</div>
      createElementVNode( // <div :class="{ active }"><p>dog</p></div>
        'div',
        { class: normalizeClass({ active: _ctx.active }) },
        [
          createElementVNode('p', null, 'dog')
        ],
        2 /* CLASS */
      ),
      createElementVNode(  // <div>{{ value }}</div>
        'div',
        null,
        toDisplayString(_ctx.value),
        1 /* TEXT */
      )
    ])
  )
}
```

### Mount 掛載

當瀏覽器 runtime render 調用這個渲染函數（Render functions） 並返回 virtual DOM tree（如下），再透過 virtual DME tree 構建真實 DOM tree，這個過程就叫做**掛載**。

```javascript
const rootVNode = {
  type: 'div',
  props: null,
  children: [
    // <div>Animals</div>
    {
      type: 'div',
      props: null,
      children: 'Animals',
    },

    // <div :class="{ active }"><p>dog</p></div>
    {
      type: 'div',
      props: {
        class: { active: _ctx.active }
      },
      children: [
        {
          type: 'p',
          props: null,
          children: 'dog',
        }
      ],
    },

    // <div>{{ value }}</div>
    {
      type: 'div',
      props: null,
      children: _ctx.value,
    }
  ],
}
```

> 以上 vnode 範例省略許多欄位，這些欄位提供效能上的幫助，減少渲染時不必要的處理。

「透過 virtual DOM tree 構建真實 DOM tree」，聽起來也很抽象耶？

其實就是 Vue 根據 vnode 來呼叫[瀏覽器的 DOM API](https://developer.mozilla.org/en-US/docs/Web/API/HTML_DOM_API)，建立真實的 DOM tree。

---

## 重點整理

### Vnode／Virtual Node（虛擬節點）

用 JavaScript 物件描述單一 DOM 節點的資料結構

### Virtual DOM（虛擬 DOM）

多個 vnode 組成的樹狀結構

### Compile（編譯）

把 Vue template 變成能夠返回 vnode object 的渲染函數

### Mount（掛載）

渲染函數返回 virtual DOM tree，透過 virtual DME tree 構建真實 DOM tree 的過程

---

寫到這裡，覺得篇幅太長，程式碼佔太多位置，可是如果沒有附上舉例的話，又很沒有畫面。

關於 Patch 和 Compiler-Informed Virtual DOM ，詳見下一篇！

* 參考資料：[渲染机制 | Vue.js](https://cn.vuejs.org/guide/extras/rendering-mechanism#rendering-mechanism)
    
* Cover credit：Gemini