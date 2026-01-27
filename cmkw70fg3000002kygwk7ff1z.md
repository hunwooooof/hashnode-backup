---
title: "Vue Patch Flag 怎麼用位元運算快速判斷動態更新？"
seoTitle: "Vue Patch Flag 是什麼？用位元運算快速判斷動態更新"
seoDescription: "了解 Vue 中使用位元運算進行更新標記的原理。"
datePublished: Tue Jan 27 2026 06:05:51 GMT+0000 (Coordinated Universal Time)
cuid: cmkw70fg3000002kygwk7ff1z
slug: vue-patch-flag
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1769493861766/2b9447b8-818b-4c83-bb4a-17ec302e5c03.png
tags: vuejs, frontend-development

---

## 前言

小的才疏學淺、孤陋寡聞，是一直到看了 [Vue 渲染機制](https://cn.vuejs.org/guide/extras/rendering-mechanism.html)的文件，才知道有**位元運算**這種工具（概念？）的存在。寫給同樣第一次看到位元運算的人，看完這篇，就能更搞懂 patchFlag 的運作原理了！

## 位元運算

在 [從模板到畫面，Vue 的渲染機制是什麼？（下）](https://wanyu.hashnode.dev/vue-template-to-dom-2#heading-2-patch-flags)的更新類型標記中，我們有提到，Vue 在編譯時會替各個元素標上更新類型的記號，並且用**位元運算**進行檢查。

到底何謂「用位元運算進行檢查」？我們先貼上 Vue 原始碼中的 [**patchFlag.ts**](https://github.com/vuejs/core/blob/main/packages/shared/src/patchFlags.ts) [程式碼片段](https://github.com/vuejs/core/blob/main/packages/shared/src/patchFlags.ts)：

```javascript
export enum PatchFlags {
  TEXT = 1,         // 0001 (1)
  CLASS = 1 << 1,   // 0010 (2)
  STYLE = 1 << 2,   // 0100 (4)
  PROPS = 1 << 3,   // 1000 (8)

  ...
  CACHED = -1
```

假設今天有一個 vnode 被標上 patchFlag = 11，我們可以先把 11 轉成二進位來看，再進行拆解。

1. 轉成二進位
    

```javascript
11 = 1011
```

2. 拆解位元
    

```javascript
1011
↑ ↑↑
8 21
```

由此可知，這個 patchFlag 對應到 `TEXT = 1`、`CLASS = 2`、`PROPS = 8`，表示這個 vnode 有**動態內容**、**動態 class**，以及**除了 class 和 style 以外的動態 props**。用人腦看一看就懂，但是用運算的怎麼算？

以下是官方文件中舉例的，簡化過的**位元運算**檢查區塊：

```javascript
if (vnode.patchFlag & PatchFlags.TEXT) {
  // Diff and update DOM
}
```

這裡的 `&` 是**位元運算的 AND**（筆者乍看一直以為是 `&&`！囧），會將兩個長度一樣的二進位數做處裡並回傳一個數字。兩個同樣的二進位都是 1，才會回傳 1。以上面的 `patchFlag = 11` 為例，運算的結果如下：

```javascript
1011   (vnode.patchFlag)
0001   (PatchFlags.TEXT)
----
0001   = 1
```

以此類推其他的 PatchFlags，經過 AND 的運算，`vnode.patchFlag` 裡面包含哪些類型的更新標記，都像照妖鏡一樣一看就知道！當確認有更新類型標記後，剩下的就是比較並更新 DOM 的事情了。是不是超酷！

---

希望此篇文章有用淺顯易懂的方式介紹 Vue 如何用位元運算快速檢查更新類型標記。[位元運算](https://zh.wikipedia.org/wiki/%E4%BD%8D%E6%93%8D%E4%BD%9C)好像又是另一個深坑<s>臭豆腐</s>，有興趣可以看看其他的位元運算方式。

* 參考資料：[**按位元與（AND）**](https://zh.wikipedia.org/wiki/%E4%BD%8D%E6%93%8D%E4%BD%9C#%E6%8C%89%E4%BD%8D%E4%B8%8E%EF%BC%88AND%EF%BC%89)、[**渲染机制 | Vue.js**](https://cn.vuejs.org/guide/extras/rendering-mechanism#rendering-mechanism)
    
* Cover credit：Gemini