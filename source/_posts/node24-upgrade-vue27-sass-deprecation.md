---
title: Node 24 升级可行性验证 — Vue 2.7 + Vue CLI 5
date: 2026-06-10 18:36:47
author: blog-author
categories:
  - 开发笔记
tags:
  - Node.js
  - Vue
  - Sass
  - 前端工程化
  - 踩坑实录
---

## 背景：一个老项目要升 Node 24

有个 Vue 2.7 + Vue CLI 5 的老项目，从建起来就没怎么升级过——Node、Vue CLI、各种 npm 依赖全是好几年前的版本。这次要做容器化部署（Kubernetes），运维那边提了个硬性前提：**先把已经 EOL（停止维护）的运行时和依赖升上去，能上 Node 24 最好**。

听起来是个例行升级，结果第一步就被卡住了。

<!-- more -->

## 真正的拦路虎：`node-sass` 在新 Node 上装不了

项目里用的是 `node-sass 4.14.1`，它底层是 **LibSass** 引擎。问题在于：

- `node-sass 4.x` **没有 Node 14 以上的预编译二进制**（prebuilt binary）；
- 想从源码编译，又依赖 `node-gyp 3.8.0` + **python2**；
- 在现代 Node 环境（Node 22 / 24）里 `npm install` **直接失败**。

也就是说，只要还抱着 `node-sass`，连干净安装都过不去。所以升级路上第一颗必须拔掉的钉子，就是把 **`node-sass`（LibSass）换成 `dart-sass`（官方现在的 `sass` 包）**。

`node-sass` 早在 2020 年就官宣弃用了，换 `dart-sass` 是迟早的事。于是先做一个「Node 24 + dart-sass」的可行性验证。

## 换完之后：冒出一大堆 SCSS deprecation 警告

工具链换成 dart-sass 后，`npm run serve` 末尾来了一句：

```text
WARNING  Compiled with 22 warnings
...
458 repetitive deprecation warnings omitted
```

22 行只是 webpack 折叠后的展示，底层实际有 **约 480 条** deprecation 警告，全部来自 `sass-loader`（也就是 SCSS 编译阶段）。

第一反应可能是「升级把项目搞坏了」。但先别慌——**这是这次最关键的认知点。**

## 关键认知：这是 warning，不是 error

把日志拉到最后，你会看到 `App running at` / `Build complete`——**编译是成功的**。进一步验证：

- dev server 正常启动，页面功能、样式、主题色都正常注入；
- **更硬的证据**：用 Node 22 和 Node 24 分别 `npm run build`，**产物的 webpack hash 逐位一致**。

这说明这些警告是**纯编译期**的 Sass 语法弃用提示，**不改变输出的 CSS，不影响运行时和页面**。换句话说：

> 它们是「将来某个大版本会移除这种写法」的预告，不是「现在出错了」。

## 根因：为什么升级前没有，升级后才冒出来？

这才是最有意思的地方。同样一份 SCSS 代码，为什么换个引擎就开始报警告？

| | 旧引擎 `node-sass`（LibSass） | 新引擎 `dart-sass` |
|---|---|---|
| 维护状态 | 2020 年起冻结、弃用 | Sass 官方当前实现 |
| 对旧语法的态度 | **从不实现弃用警告**，旧写法静默通过 | **主动**对将在 Dart Sass 2.0 / 3.0 移除的语法发警告 |

所以真相是：**这些旧语法在第三方库里早就存在了**，只是 LibSass 一直「睁一只眼闭一只眼」。这次迁移到 dart-sass，等于把这些**历史包袱「暴露」成了可见警告**。

它不是新引入的 bug，也跟 Node 24 本身没关系——**纯粹是引擎换代的预期副作用**。

而且，约 **95% 以上的警告来自一个第三方 UI 组件库**（一个已经 EOL 的 Vue 2 时代组件库）。项目全量编译了它的主题 SCSS，而它的源码还是老式的 `@import` / 全局 `mix()` / `/` 除法写法。真正属于自己项目代码的，只有 **1 行** `@import`。

## 五类警告长什么样

dart-sass 的弃用警告主要是这 5 类，都很典型，老项目升级几乎必遇：

| # | 类别 | 弃用提示关键词 | 将在哪个版本移除 |
|---|---|---|---|
| 1 | Legacy JS API | `The legacy JS API is deprecated` | Dart Sass 2.0 |
| 2 | `@import` 规则 | `Sass @import rules are deprecated` | Dart Sass 3.0 |
| 3 | 全局内建函数 | `Global built-in functions are deprecated`（如 `mix()` → `color.mix`） | Dart Sass 3.0 |
| 4 | 无单位数当百分比 | `Passing a number without unit % is deprecated` | — |
| 5 | `/` 当除法 | `Using / for division outside of calc() is deprecated` | Dart Sass 2.0 |

第 1 类（legacy JS API）属于**工具链**——`sass-loader` 默认还在调用 dart-sass 的旧版 `render` API。其余 4 类几乎全来自那个第三方组件库的主题 SCSS。

## 影响评估：现在到底要不要管它？

| 维度 | 结论 |
|---|---|
| 编译是否成功 | ✅ 是，warning 不阻断 |
| 产物正确性 | ✅ Node 22/24 构建 hash 一致 |
| 运行时 / 页面 | ✅ 零影响（纯编译期提示） |
| 什么时候会变成 error | 只有当你把 `sass` 升到 **2.0 / 3.0** 大版本时 |
| 当前风险 | 🟢 低——`package.json` 把 `sass` 锁在 `^1.x`，它永远停在 1 字头，这些条目长期是 warning，**不会自己变成 error** |

结论很清楚：**短期内完全可以不管它。**

## 解决方案：能根治吗？分层处理

### 能不能彻底根治？

老实说：**在自己仓库里根治不了**。因为 95% 的警告来自那个已经 EOL 的第三方组件库，上游不会再修了。**唯一的彻底根治路径，是迁移到它的新版本（Vue 3 生态的那个组件库）**——那是另一个大工程，不该塞进一次工具链升级里。

所以现实做法是分层：

### 方案 0：不处理（最省事）

warning 无害，`sass` 锁 1.x，啥也不动。符合「只升工具链、不动业务代码」的原则。

### 方案 1：`quietDeps` 静音第三方噪声（推荐降噪手段）

dart-sass 提供了 `quietDeps`，专门**静音来自 `node_modules` 依赖的弃用警告**，但保留你自己代码的。在 `vue.config.js` 里：

```javascript
module.exports = {
  css: {
    loaderOptions: {
      scss: {
        // 静音来自 node_modules 第三方依赖的弃用警告
        // 注意：保持默认 legacy API，不要设 api:'modern'（见下方的坑）
        sassOptions: { quietDeps: true }
      }
    }
  }
}
```

实测效果：警告从 **20 条降到 1 条**（剩的那 1 条是自己项目的 `@import`）。如果想连这条也静音，可以再加 `silenceDeprecations: ['import', 'global-builtin', 'color-functions', 'slash-div', 'legacy-js-api']`，直接降到 **0**。只改构建配置，不碰业务代码，build 照样成功。

### ⚠️ 一个反直觉的坑：`api: 'modern'` 不是 drop-in

网上很多人会说「切到现代 Sass API（`api: 'modern'`）就能根治 legacy-js-api 警告」。**但在这个项目里，实测会直接让 build 失败：**

```text
Can't find stylesheet to import: @import "~some-ui-lib/.../index.scss"
```

根因：**现代 Sass API 不再支持 webpack 的 `~` 前缀解析**。项目里所有 `@import "~xxx/..."` 这种写法全部解析不了。要切现代 API，得先把项目内所有 `~` 前缀的 SCSS import 改成 bare import 再配 `loadPaths`——改造面不小。

所以结论是：**别为了消一条 legacy-js-api 警告去切现代 API**，得不偿失。真要消，用上面的 `silenceDeprecations: ['legacy-js-api']` 静音就行。

## 经验总结

这次「升 Node 24」表面是版本号的事，实际踩下来有几条可复用的经验：

1. **老项目升 Node，最先炸的往往是 `node-sass`。** 它没有新 Node 的预编译二进制、还要 python2，现代环境装不上。遇到就直接换 `dart-sass`，别挣扎。

2. **`node-sass → dart-sass` 之后冒出的一堆 SCSS deprecation，绝大多数不是你的锅。** LibSass 一直静默放过的旧语法，被 dart-sass 如实暴露出来，源头通常是某个 EOL 的第三方库。

3. **分清 warning 和 error。** 编译成功、产物 hash 一致、运行时无影响——这种就是预告性质的弃用提示，不必为了「日志干净」过度反应。把 `sass` 锁在 1.x，它就不会自己变 error。

4. **想要日志清爽，用 `quietDeps` 静音第三方，而不是去改第三方代码或硬切现代 API。** 一行构建配置解决 95% 的噪声。

5. **`api: 'modern'` 在带 `~` 前缀 import 的老项目里不是 drop-in 替换**，盲目切会让 build 失败。升级前先确认改造成本。

说到底，**升级的目标是「脱离 EOL、为后续铺路」，不是「消灭每一条 warning」**。分清哪些必须现在解决、哪些可以静音、哪些留给未来的大版本迁移，比一股脑追求「零警告」要务实得多。
