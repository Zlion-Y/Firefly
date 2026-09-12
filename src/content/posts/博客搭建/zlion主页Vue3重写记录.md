---
title: 参考 imsyy/home 重写的个人主页：Vue 3 + Vite 轻量实现与踩坑记录
published: 2026-09-12
updated: 2026-09-12
description: 从 imsyy/home 出发重写一个自己的导航主页：Vue 3 + Vite 单依赖实现毛玻璃 Bento 布局、FluentPlayer 同款卡片 3D 动效、自定义光标涟漪、高德天气精确到市区与平滑温度曲线，构建产物 gzip 仅 35KB，GitHub + Vercel 免费部署
image: ./preview.png
tags: [Vue3, Vite, 个人主页, 前端]
category: 博客搭建
draft: false
pinned: false
slug: zlion-home-vue3
author: 世琰
---

> 本文记录给自己的导航站 [zlion.top](https://github.com/Zlion-Y/zlion-home) 从选型到上线的完整过程。
> **参考项目：** [imsyy/home](https://github.com/imsyy/home)（MIT License，已存档）
> **技术栈：** Vue 3.5 + Vite 7，唯一运行时依赖是 vue 本身
> **部署：** GitHub + Vercel，零配置

---

## 一、为什么不用原版

[imsyy/home](https://github.com/imsyy/home) 是 4.6k star 的知名主页项目，但直接拿来用有两个问题：

1. **仓库已存档**（read-only），作者不再维护；
2. **依赖偏重**：Element Plus、Pinia、APlayer、Swiper 一应俱全，对一个导航页来说 surplus 太多。

我的需求其实很简单：**站点导航 + 一言 + 时钟 + 天气**，于是决定参考它的布局思路，用 Vue 3 + Vite 重写一份，最终构建产物 gzip 仅 ~35KB（原版含组件库通常在数百 KB 量级）。

## 二、布局：从手绘草图到 Bento 网格

最终布局是拿平板画草图定稿的：

```
左列（5fr）                右列（7fr）
Z 徽标 + zlion.top         一言        | 时钟
问候语卡片                  天气长卡（跨 2 列）
「关于我」标签卡             网站列表（3 列小卡）
```

实现上就是两层 Grid：

```css
.container {
  display: grid;
  grid-template-columns: 5fr 7fr;
  /* 视口富余时垂直居中，内容超高时退化为顶对齐（防裁顶） */
  align-content: safe center;
}

.bento {
  display: grid;
  grid-template-columns: 1fr 1fr;
  grid-auto-rows: 150px;
  gap: 24px;
}

.bento .weather { grid-column: span 2; } /* 天气长卡 */
```

几个布局上的经验：

- **卡片高度全部写死在网格行上**（`grid-auto-rows`），而不是让内容撑开——否则一言长短句切换会把整行顶得跳一下；
- 打字机文案区**固定两行高度**（`min-height: 3.6em`），同理；
- `align-content: safe center` 是关键技巧：直接写 `center` 在小屏内容溢出时会裁掉顶部，加 `safe` 前缀让它溢出时自动退化为 `start`。

所有个性化集中在一个 `config.js`：站名、问候语、打字机标语、社交链接、网站列表、建站日期，改完 push 即自动重新部署。

## 三、动效：hover 才触发的卡片 3D 倾斜

第一版我做成了"鼠标在页面任意位置，所有卡片持续跟随倾斜"，实际体验反而显得闹腾。最终改成和我的音乐播放器 [FluentPlayer](https://github.com/Zlion-Y/FluentPlayer) 播放页封面一致的手感：**鼠标移入卡片才倾斜，按卡内相对位置变换，移出归零回弹**：

```ts
const maxRotate = 12;
tiltRotateY = ((x - centerX) / centerX) * maxRotate;
tiltRotateX = -((y - centerY) / centerY) * maxRotate;
// perspective(1000px) + scale3d(1.02)，transition 240ms cubic-bezier(0.22,1,0.36,1)
```

配合一层 `mix-blend-mode: overlay` 的径向渐变光泽跟随鼠标，就是熟悉的"果冻玻璃"质感。

这里踩了本次最大的一个坑：**CSS 动画的 `forwards` 填充优先级高于内联样式**。最初进场动画用 `animation: rise .8s both`，动画播完后 `transform: none` 被永久锁定，JS 写入的倾斜 transform 完全失效。解法是进场改用 `transition`（配合 `transition-delay` 做级联进场），JS 在 1.5s 后接管 transform；接管时同时用内联样式覆盖 `.glass` 上 0.5s 的 transform 过渡，否则逐帧倾斜会被过渡拖成慢动作。

另外 `.glass` 因为圆角裁切设了 `overflow: hidden`，悬停提示气泡从上方弹出会被裁掉一半——把气泡改为向下弹出解决。

## 四、自定义光标 + 涟漪

进入页面后隐藏系统光标（`cursor: none`），替换为：

- **中心圆点**：即时跟随（直接 transform）；
- **外圈**：`lerp 0.18` 缓动跟随，制造"拖尾"的物理感；悬停可点击元素时外圈变色放大；
- **移动涟漪**：每移动 42px 泛起一圈小涟漪，`mousedown` 泛大涟漪；

```js
function loop() {
  rx += (mx - rx) * 0.18;
  ry += (my - ry) * 0.18;
  ring.style.transform = `translate(${rx}px, ${ry}px)`;
  requestAnimationFrame(loop);
}
```

触屏设备（`hover: none`）和 `prefers-reduced-motion` 下整套系统自动禁用。

## 五、天气卡：精确到市/区 + 平滑温度曲线

用的是高德免费 Web 服务 API，三个请求并行：

1. `v3/ip`：IP 定位。**注意它通常只给到省级**（比如"湖北省"）；
2. 拿定位返回的 `rectangle`（范围两角坐标）取**中点**，调 `v3/geocode/regeo` 逆地理编码，解析出市/区级的城市名和 adcode——这样天气就能显示"武汉"而不是"湖北"；
3. `v3/weather/weatherInfo`：`extensions=base` 取实况（温度/湿度/风向），`extensions=all` 取未来 4 天预报画曲线。

两个 API 坑：

- **预报数组在 `forecasts[0].casts`**，顶层 `forecasts` 只是包装数组，取错层级曲线永远是空的；
- `city` 字段在直辖市返回的是**空数组**而不是字符串，需要兼容回退到 `province`。

温度曲线用 Catmull-Rom 转三次贝塞尔把折线变平滑：

```js
for (let i = 0; i < pts.length - 1; i++) {
  const p0 = pts[i - 1] || pts[i], p1 = pts[i], p2 = pts[i + 1], p3 = pts[i + 2] || p2;
  d += ` C ${p1.x + (p2.x - p0.x) / 6} ${p1.y + (p2.y - p0.y) / 6}, ` +
       `${p2.x - (p3.x - p1.x) / 6} ${p2.y - (p3.y - p1.y) / 6}, ${p2.x} ${p2.y}`;
}
```

纯内联 SVG，零依赖。另外高德接口在测试时频繁调用会间歇限流，所以做了失败 3 秒重试一次、仍失败整卡隐藏、网格自动补位的容错。

![主页最终效果：一言/时钟/天气长卡与温度曲线](./preview.png)

## 六、部署

GitHub 建仓库后 Vercel 导入即可，Framework 自动识别 Vite，零配置。

唯一要注意的是天气 Key 用了 `VITE_` 前缀：**Vite 的 `VITE_` 前缀变量就是设计为暴露给浏览器的**，Vercel 会警告"该值会公开"，这里属于有意为之（前端直连高德 API），在 Vercel 的环境变量类型里选 Config 即可。个人用量在高德免费配额内完全够用。

## 七、结语

仓库在 [Zlion-Y/zlion-home](https://github.com/Zlion-Y/zlion-home)，欢迎参考/自用。回头看这次重写的收获：

1. **参考布局思路而不是搬代码**——原版的重依赖一个都没带进来；
2. **动效手感直接抄成熟项目**（FluentPlayer 的 tilt 参数），比凭感觉调快得多；
3. 排查问题时印象最深的两课：CSS `forwards` 会压住内联 transform；本地 preview 反复重启会残留进程占端口 + 浏览器缓存 index.html，两者叠加会造成"改了代码没效果"的假象——验证前先确认自己看到的是最新构建。
