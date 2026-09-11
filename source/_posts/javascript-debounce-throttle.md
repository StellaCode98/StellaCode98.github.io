---
title: 手写防抖与节流：30 行核心代码，讲清两件事
date: 2026-09-10 10:00:00
description: 防抖和节流都是「高频事件降频」的手段：防抖只认最后一次——停止触发 wait 毫秒后才执行；节流认时间窗口——无论触发多频繁，固定周期内最多执行一次。本文手写 debounce / throttle 的基础版、立即执行版与合并版，一张表说清该用哪个。
categories:
  - [前端基础, JavaScript]
tags:
  - JavaScript
  - 性能优化
---

`scroll`、`resize`、`input`、`mousemove` 这类事件一秒能触发几十上百次，处理器跟不上就是卡顿。防抖和节流做的事情一样——**降频**，但策略相反：

> **防抖（debounce）：等你停了我再做。** 每次触发都重新计时，只有安静了 `wait` 毫秒才执行，持续触发期间一次都不跑。
> **节流（throttle）：再急也按固定节奏做。** 不管触发多频繁，每个 `wait` 窗口内最多执行一次。

电梯类比：防抖是「有人进来就重新等，没人再进才关门」；节流是「观光车每 10 分钟发一班，到点就走」。

<!-- more -->

## 一、手写防抖

### 基础版

核心就一招：**用闭包存住定时器，每次触发先把上一个清掉**。

```js
function debounce(fn, wait) {
  let timer = null;
  return function (...args) {
    if (timer) clearTimeout(timer);   // 上一次作废，重新计时
    timer = setTimeout(() => {
      fn.apply(this, args);           // this 指向事件目标，参数透传
    }, wait);
  };
}

// 用法
input.addEventListener('input', debounce(search, 300));
```

两个容易被问的细节：

- **外层必须是普通 `function`**：`this` 才能拿到绑定事件的元素；内层用箭头函数继承这个 `this`。
- **`fn.apply(this, args)`**：不写的话 `fn` 里的 `this` 是 `window`、`event` 也丢了。

### 完善版：支持立即执行 + 取消

搜索框更理想的体验是**第一次输入立刻搜，之后的连击被防抖吞掉**——这就是 `immediate` 选项。组件卸载时还得能取消等待中的任务，防止「人已下车还执行回调」。

```js
function debounce(fn, wait, immediate = false) {
  let timer = null;

  const debounced = function (...args) {
    if (timer) clearTimeout(timer);

    if (immediate) {
      const callNow = !timer;                    // 没有等待中的任务 → 这次立刻执行
      timer = setTimeout(() => (timer = null), wait); // 定时器只用来"锁"住冷却期
      if (callNow) fn.apply(this, args);
    } else {
      timer = setTimeout(() => fn.apply(this, args), wait);
    }
  };

  debounced.cancel = () => {                     // 取消：清掉定时器并复位
    clearTimeout(timer);
    timer = null;
  };
  return debounced;
}
```

`immediate` 分支的巧妙之处：定时器到点后**不执行 fn，只把 `timer` 置空**，它退化成一把「冷却锁」——锁着的时候（`timer` 非空）不执行，锁开了才放行下一次。

## 二、手写节流

### 时间戳版：首次立即执行，末尾不补

```js
function throttle(fn, wait) {
  let prev = 0;                          // 上次执行时间，0 保证首次立即执行
  return function (...args) {
    const now = Date.now();
    if (now - prev >= wait) {            // 距上次执行够久了，放行
      fn.apply(this, args);
      prev = now;
    }
  };
}
```

缺点：停止触发前，最后一次可能刚好卡在窗口里，不会被补执行。

### 定时器版：首次延迟执行，末尾补一次

```js
function throttle(fn, wait) {
  let timer = null;
  return function (...args) {
    if (timer) return;                   // 已有任务在排队，直接丢弃本次
    timer = setTimeout(() => {
      fn.apply(this, args);
      timer = null;                      // 释放，允许排下一个任务
    }, wait);
  };
}
```

缺点：第一次也要等 `wait` 才执行。

### 合并版：首尾都执行

用「剩余时间」把两种策略拼起来：窗口已过就立即执行，还在窗口内就排一个 `remaining` 毫秒后的兜底任务。

```js
function throttle(fn, wait) {
  let timer = null;
  let prev = 0;
  return function (...args) {
    const remaining = wait - (Date.now() - prev);

    if (remaining <= 0) {                // 窗口已过：立即执行
      clearTimeout(timer);
      timer = null;
      fn.apply(this, args);
      prev = Date.now();
    } else if (!timer) {                 // 窗口内：补一次尾部执行
      timer = setTimeout(() => {
        fn.apply(this, args);
        prev = Date.now();
        timer = null;
      }, remaining);
    }
  };
}
```

## 三、一张表选型

假设持续触发 1 分钟、`wait` 为 1 秒：

| | 防抖 debounce | 节流 throttle |
| --- | --- | --- |
| 执行时机 | 停止触发 `wait` 后 | 固定周期到了就执行 |
| 1 分钟内的执行次数 | **1 次**（只认最后一次） | **约 60 次**（稀释频率） |
| 典型场景 | 搜索联想、`resize` 后重算布局、表单校验、防重复提交 | 滚动加载/吸顶、`mousemove` 跟随、按钮防连点、视频进度上报 |

判断口诀：**只关心最终结果的用防抖（结果型），过程中必须周期响应的用节流（过程型）。** 拿不准就想电梯和观光车。

<!-- ## 四、面试速答清单

- 防抖：闭包 + `clearTimeout` 重置定时器；节流：时间戳差值判断或定时器占位。
- 三个共性细节：普通 `function` 保留 `this`、`apply` 透传 `args`、返回新函数而非原地改。
- 防抖进阶：`immediate`（首刷 + 冷却锁）、`cancel`（组件卸载时调用）。
- 节流两版的差异：时间戳版首执行尾不补，定时器版首延迟尾补一次，合并版兼得。
- 生产环境直接用 `lodash` 的 `debounce` / `throttle`，它们还处理了 `maxWait`、`leading/trailing` 开关等边界。 -->
