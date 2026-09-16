---
title: 重写了一份个人主页：Vue 3 + Vite 毛玻璃 Bento，还接上了洛雪音源
published: 2026-09-12
updated: 2026-09-16
description: 分享一下给自己重写的导航主页：Vue 3 + Vite，唯一运行时依赖是 vue，构建产物 gzip 约 60KB。毛玻璃 Bento 布局 + 全配置驱动的二级「探索更多」面板（新闻/热榜/音乐/Epic 限免/历史上的今天/站点监控），免 Key 天气定位到县级；音乐播放器接上洛雪音源——自定义音源脚本跑在 serverless 函数里，VIP 曲也能听到完整版。零环境变量部署到 Vercel，fork 改一个 config.js 就能变成你自己的主页
image: ./preview.jpg
tags: [Vue3, Vite, 个人主页, 前端, 洛雪音源, serverless]
category: 博客搭建
draft: false
pinned: false
slug: zlion-home-vue3
author: 世琰
---

> **在线访问：** [www.zlion.top](https://www.zlion.top) ｜ **开源仓库：** [Zlion-Y/homepage](https://github.com/Zlion-Y/homepage)
> 布局思路最初参考了 [imsyy/home](https://github.com/imsyy/home)（MIT License，已存档），在此致谢——除此之外没有搬它的代码，依赖也全部换掉了。

![主页效果：壁纸背景 + 毛玻璃卡片](./preview.jpg)

## 为什么自己写一份

想要的东西其实很简单：**站点导航 + 一言 + 时钟 + 天气 + 音乐**。imsyy/home 的布局我很喜欢，但它已经归档不再维护，而且 Element Plus、Pinia、APlayer、Swiper 一整套对一个导航页来说太重了。于是用 Vue 3 + Vite 重写了一份：**唯一运行时依赖就是 vue 本身**，构建产物 gzip 约 60KB（JS 52KB + CSS 9KB）。

## 功能一览

- **毛玻璃 Bento 布局**：一言、时钟、天气长卡、网站列表小卡自由拼装，移动端自动降级单列
- **一言**：Hitokoto API，失败自动兜底本地语句，点卡片换一句
- **实时时钟**：农历 + 节日倒数
- **实时天气长卡**：uapis 聚合接口，**免 Key**、按访客 IP 自动定位到县级（IPv6 正常），湿度/体感/AQI、未来几天高低温双曲线，可固定城市
- **博客更新卡**：自动拉取博客 RSS 展示最新 3 篇文章，卡头带 GitHub / 邮箱等社交图标
- **网站列表**：一行 3 个紧凑卡片，`config.js` 里加站点自动扩展
- **二级「探索更多」面板**：点左上角 Logo 进入，桌面一屏六卡——
  - 每日新闻（60s API）
  - 多平台热榜（微博/B站/V2EX/IT之家等，平台与顺序可配置）
  - 音乐播放器（下一节重点）
  - Epic 限免
  - 程序员历史上的今天
  - 站点监控（访客浏览器直连探测自己各站点，绿红点 + 响应耗时，60 秒自动刷新，无需第三方监控服务）
- **动效**：卡片悬停 3D 倾斜 + 光泽、自定义圆点光标 + 点击涟漪、卡片错峰浮起
- **细节**：极光渐变背景随昼夜变色、支持自定义壁纸/随机壁纸 API、页脚建站天数、站名 7 款手写字体一键切换

所有卡片都能在 `config.js` 里开关与排序。

## 音乐播放器：多源自愈 + 洛雪音源

### 播放器本身

- 多 Meting 源**并发竞速**拉歌单，谁先返回用谁，单源故障无感
- 每首歌自动在多个源之间探测可用播放链接，断链/过期自愈，全败自动跳下一首
- 歌单本地缓存 6 小时，进面板秒开；预载但不自动出声，点了播放才响
- 歌词同步滚动居中，点歌词行跳转进度

### 接上洛雪音源（serverless）

公共 Meting 接口有个绕不过去的边界：**VIP / 版权受限曲目拿不到完整直链**——它给的是一段能正常播的 30 秒试听片段，不报错、很难察觉。仓库自带一份 serverless 版洛雪音源解析解决这件事：

- **自定义音源脚本跑在 Vercel 函数里**（`api/` + `lib/`，跟主页一起部署），多音源按成绩分波对冲、赢家一出即掐断其余、直链探活、连续失败熔断
- 前端调**同源**的 `/api/url`：不需要额外域名、证书、CORS 配置，也不用再养一台服务器
- 直链缓存在 CDN 边缘（15 分钟），**命中缓存的请求根本不进函数**，不消耗调用次数
- 解析不到时自动降级回 Meting，开着也不影响原来能用的情况

`config.js` 里两个开关：

```js
musicSource: "proxy",    // "meting" = 只走公共 Meting；"proxy" = 走自带的 serverless 解析
musicQuality: "320k",    // 128k / 320k / flac / flac24bit
```

音源脚本放 [`sources/`](https://github.com/Zlion-Y/homepage/tree/main/sources) 一起部署；也可以配 `SOURCE_URLS` 环境变量指向在线脚本——改完远端脚本请求一次 `/api/health?refresh=1`，函数立刻重装全部音源，**连重新部署都不用**。部署后打开 `/api/health` 能看到装上了哪些音源、各自的平台与失败原因。

> 提醒：函数默认跑在香港（`hkg1`，离国内接口最近），实测与国内出口的成功率一致；但这类「直链解析」本身在灰区，建议自用、别公开分发。

## 部署

### 平台托管（推荐）

Vercel / Netlify / Cloudflare Pages 均可，Framework 选 `Vite`，根目录 `./`、输出目录 `dist`、构建命令 `npm run build`，**零环境变量**（免 Key 接口的好处）。

[![Deploy with Vercel](https://vercel.com/button)](https://vercel.com/new/clone?repository-url=https://github.com/Zlion-Y/homepage&project-name=zlion-homepage&repository-name=zlion-homepage)

### 本地开发

```bash
git clone https://github.com/you-github-name/homepage.git
cd homepage
npm install
npm run dev     # http://localhost:5173，配置热更新
```

## 全部配置化：只改 `src/config.js`

个性化集中在一个文件，改完 push 即自动重新部署。常用的几项：

| 配置项 | 说明 |
| --- | --- |
| `siteName` / `pageTitle` / `greet` / `desc` / `motto` | 站名、标签页标题、问候语、一句话介绍、打字机轮换标语 |
| `siteFont` | 站名手写体，内置 7 款开源字体一键切换 |
| `logo` | 留空用内置「Z」徽章，填图片路径/URL 整体替换 |
| `siteStart` / `author` / `repo` | 建站日期（页脚运行天数）、署名、署名链接 |
| `homeCards` | 主页各卡片开关（greet / blog / hitokoto / clock / weather / siteLinks） |
| `panelCards` | 二级面板卡片排列：数组顺序 = 排列顺序，删掉某项 = 隐藏该卡 |
| `musicPlaylist` / `musicSource` / `musicQuality` | 歌单 ID、直链来源、音质 |
| `weatherCity` | 天气城市 adcode，留空 = 按访客 IP 自动定位 |
| `bgApi` | 背景：留空极光渐变 / 放 `public/images/background.jpg` / 填随机壁纸 API |
| `hotPlatforms` | 热榜平台与顺序 |
| `siteMonitors` | 站点监控卡的目标站点列表 |
| `socialLinks` / `siteLinks` | 社交图标、网站列表 |

换成你自己的站点，改 `siteLinks` 和 `socialLinks` 就行。

## TODO

- [ ] 更多卡片支持（欢迎 PR / Issue 讨论）
- [ ] 卡片位置自定义（拖拽排序 / 布局记忆）
- [ ] 音乐卡 AMLL 歌词动效（逐字点亮、弹性动画、灵感封面背景）

## 致谢

- [imsyy/home](https://github.com/imsyy/home)——布局灵感来源；
- [CuteLeaf/Firefly](https://github.com/CuteLeaf/Firefly)——音乐播放器多源自愈设计参考；
- [uapis.cn](https://uapis.cn/)、[60s API](https://github.com/vikiboss/60s)、[hitokoto.cn](https://hitokoto.cn/)——免 Key 数据接口；
- [Lucide](https://lucide.dev/)——图标。

项目为 vibe coding 实现，若侵犯了您的权益请联系我删除。觉得有用的话，去 [仓库](https://github.com/Zlion-Y/homepage) 点个 Star 就是对我最大的支持～
