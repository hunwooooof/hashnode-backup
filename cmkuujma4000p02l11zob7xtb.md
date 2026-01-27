---
title: "從模板到畫面，Vue 的渲染機制是什麼？（下）"
seoTitle: "從模板到畫面，Vue 的渲染機制是什麼？（下）"
seoDescription: "Explore Vue's advanced rendering with Virtual DOM, Compiler-Informed Virtual DOM, and Patch for optimized component updates"
datePublished: Mon Jan 26 2026 07:29:05 GMT+0000 (Coordinated Universal Time)
cuid: cmkuujma4000p02l11zob7xtb
slug: vue-template-to-dom-2
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1769411569113/7c95044c-d102-4a0a-90f3-691842f4da2c.png
tags: vuejs, frontend-development

---

如果還不知道 Virtual DOM、vnode、Compile 以及 Mount，可以先看上篇：  
[Vue 工廠 ── 從模板到畫面，Vue 的渲染機制是什麼？（上）](https://wanyu.hashnode.dev/vue-template-to-dom-1)

## 前言

介紹完 Vue 在編譯以及掛載時做了什麼，接下來要介紹 Vue 更新畫面時有多麼鬼靈精怪，像變魔術一樣，提升 Virtual DOM 的性能。看完文件，真的非常讚歎欽佩尤大大⋯⋯。

## 依賴更新了，畫面也要更新！這就是 Patch。

當使用者與網頁互動，響應式狀態發生變化時，Vue 會重新執行對應 component 的 render function，產生新的 vnode tree。新的 vnode tree 與舊的 vnode tree 進行比較，並把變更更新到真的 DOM tree。

「新的 vnode tree 與舊的 vnode tree 進行比較」聽起來很正常，但如果裡面改變的 props 其實不多呢？明明只有 某幾個 props 會變，卻還是在 runtime 逐一檢查了其他不可能改變的 props ，才知道要不要更新 DOM，是不是聽起來很多餘呢？

放心，Vue 為了避免這些無用的事情發生，在 Vue 3 帶入了新的概念：**Compiler-Informed Virtual DOM**。簡單地說，Vue 3 在編譯時，像貼標籤紙一樣，把「vnode 裡面哪部分是動態的」都做上了記號。所以，什麼是 Compiler-Informed Virtual DOM？

## Compiler-Informed Virtual DOM

Compiler-Informed Virtual DOM，官網直翻為「帶編譯時訊息的虛擬 DOM」。顧名思義，這些虛擬 DOM 的身上都帶著編譯時附加上去的訊息，而負責進行新舊 Virtual DOM 比較的 function 一看就知道：這個只需要看 class 有沒有變！那個只需要看 innerHTML 有沒有更新！

以下就來介紹幾個 Vue 官網有列舉的，在編譯時優化 Vurtual DOM 運行效能的方法：

### 1\. Cache Static 緩存靜態內容

```xml
<div>
  <div>dog</div> <!-- 需缓存 -->
  <div>cat</div> <!-- 需缓存 -->
  <div>{{ dynamic }}</div>
</div>
```

針對完全靜態的內容，對應的 vnode 在首次渲染時就會被緩存起來。後續重新渲染時，會直接使用已經緩存的 vnode，不需要在 patch 時重新創建和比對，他們的 diffing 都會被跳過。

```javascript
import { createElementVNode as _createElementVNode, createCommentVNode as _createCommentVNode, toDisplayString as _toDisplayString, openBlock as _openBlock, createElementBlock as _createElementBlock } from "vue"

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (_openBlock(), _createElementBlock("div", null, [
    _cache[0] || (_cache[0] = _createElementVNode("div", null, "dog", -1 /* CACHED */)),
    _createCommentVNode(" cached "),
    _cache[1] || (_cache[1] = _createElementVNode("div", null, "cat", -1 /* CACHED */)),
    _createCommentVNode(" cached "),
    _createElementVNode("div", null, _toDisplayString(_ctx.dynamic), 1 /* TEXT */)
  ]))
}
```

在上方的模板編譯預覽中可以看到其中一行：

`_cache[0] || (_cache[0] = _createElementVNode(...))`

初次渲染時，若還沒有 `_cache[0]`，就會執行一次 `_createElementVNode()` ，但當 `_cache[0]` 有了值，就不用再跑一次 `_createElementVNode()`。

### 2\. Patch Flags 更新類型標記

在模板裡面，哪些元素有動態綁定，不是什麼大祕密，編譯時都知道！

```xml
<!-- 只有 class 動態綁定 -->
<div :class="{ active }"></div>

<!-- id 和 value 動態綁定 -->
<input :id="id" :value="value">

<!-- 只有子節點動態綁定 -->
<div>{{ dynamic }}</div>
```

這些模板編譯成渲染涵數的同時，也會被 Vue 標上記號，紀錄下各個元素會更新的類型：

```javascript
import { normalizeClass as _normalizeClass, createElementVNode as _createElementVNode, toDisplayString as _toDisplayString, Fragment as _Fragment, openBlock as _openBlock, createElementBlock as _createElementBlock } from "vue"

export function render(_ctx, _cache, $props, $setup, $data, $options) {
  return (_openBlock(), _createElementBlock(_Fragment, null, [
    _createElementVNode("div", {
      class: _normalizeClass({ active: _ctx.active })
    }, null, 2), // <--- HERE
    _createElementVNode("input", {
      id: _ctx.id,
      value: _ctx.value
    }, null, 8, ["id", "value"]), // <--- HERE
    _createElementVNode("div", null, _toDisplayString(_ctx.dynamic), 1) // <--- HERE
  ], 64)) // <--- HERE
}
```

看到裡面的數字 `2`、`8`、`1`，以及最外層還有一個 `64` 了嗎？看不懂沒關係，我們先來看 Vue 原始碼中的 [patchFlag.ts](https://github.com/vuejs/core/blob/main/packages/shared/src/patchFlags.ts) 程式碼片段，他定義了這些數字的意思：

```javascript
export enum PatchFlags {
  /**
   * Indicates an element with dynamic textContent (children fast path)
   */
  TEXT = 1,

  /**
   * Indicates an element with dynamic class binding.
   */
  CLASS = 1 << 1,

  /**
   * Indicates an element that has non-class/style dynamic props.
   * Can also be on a component that has any dynamic props (includes
   * class/style). when this flag is present, the vnode also has a dynamicProps
   * array that contains the keys of the props that may change so the runtime
   * can diff them faster (without having to worry about removed props)
   */
  PROPS = 1 << 3,

  /**
   * Indicates a fragment whose children order doesn't change.
   */
  STABLE_FRAGMENT = 1 << 6,
```

> `<<` 稱作**位元左移運算子**，是指將二進位數的每一位向左移動指定的位數。通常 x &lt;&lt; n 等於 x×2ⁿ。
> 
> 因此 1 &lt;&lt; 1 可以想成 2¹＝2；1 &lt;&lt; 6 可以想成 2⁶＝64。

回到一開始 HTML 模板的舉例，三個元素，分別是**動態 class**、**動態 id & value**、**動態子節點**。對應到的 Patch Flag 分別是 `CLASS = 1 << 1`、`PROPS = 1 << 3`、`TEXT = 1`，因此被標記了`2`、`8`、`1`。

因為有了這些像作弊一樣的更新標記，當運行到這個元素時，只會檢查這些 props 是否有更新，不會沒事去檢查其他根本不可能更新的 props。這種檢查方式，稱作**位元運算**。

> 延伸閱讀：[Vue Patch Flag 怎麼用位元運算快速判斷動態更新？](https://wanyu.hashnode.dev/vue-patch-flag)

（那最外層標記的 `64` 是什麼意思呢？可以找到 patchFlags 對 64 的定義，再自己想想看🧠😆）

### 3\. Tree Flattening 樹結構打平

在介紹 Tree flattening 之前，我們要先介紹「block」這個概念。Vue 把一個內部結構穩定（子結構在同一個更新週期內不會改變順序或數量，例如使用 `v-for` 或 `v-if` 指令）的 vnode 子樹稱之為 block。在上面的範例程式碼裡面，我們都可以看到 render function 調用了 `_createElementBlock()` 這個 method，它便是回傳了一個有 block 概念的區塊。

我們再次用第一個範例來說明，並且再多加一個動態區塊：

```xml
<div>
  <div>dog</div> <!-- 不用追蹤 -->
  <div>cat</div> <!-- 不用追蹤 -->
  <div>{{ dynamic }}</div> <!-- 要追蹤 -->
  <div>
    <div :class="dynamicClass">pig</div> <!-- 要追蹤 -->
  </div>
</div>
```

上述模板裡，雖然有使用到動態參數，但仍可被稱為一個「內部結構穩定」的區塊，因為子節點皆是固定存在的，不會消失。而 Vue 編譯完這塊模板後，會把所有需要動態更新的後代節點，收集成一個扁平陣列如下：

```javascript
dynamicChildren = [
  VNode(div, TEXT),
  VNode(div, CLASS),
]
```

有了上方 `dynamicChildren` 陣列，當這個 block 需要重新渲染時，只需要訪問 `dynamicChildren` 就能知道可能改變的地方，大幅減少了虛擬 DOM 需要遍歷比對的節點數量。

---

## 重點整理

### Patch

響應式狀態更新渲染函數重新執行並比對 Virtual DOM 然後更新到 DOM 的過程

### Compiler-Informed Virtual DOM

透過在編譯時對渲染函數加上信息，來降低比對負擔，增加效能。

---

這篇渲染機制越看越覺得自己前一次其實沒看懂，反覆看了很多次，每次都能看到新重點，深覺自己還有很多不懂的地方，如果有講解不順的地方，還請見諒！

* 參考資料：[**渲染机制 | Vue.js**](https://cn.vuejs.org/guide/extras/rendering-mechanism#rendering-mechanism)
    
* Cover credit：Gemini