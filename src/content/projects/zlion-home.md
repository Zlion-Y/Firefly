---
title: "zlion-home"
slug: zlion-home
published: 2026-09-12
draft: false
description: "参考 imsyy/home 重写的个人导航主页：Vue 3 + Vite，唯一运行时依赖是 vue 本身。毛玻璃 Bento 布局 + 全配置驱动的二级面板（新闻/热榜/音乐播放器/Epic 限免/历史上的今天/站点监控），免 Key 天气定位到县级；音乐播放器接上洛雪音源——自定义音源脚本跑在 serverless 函数里，VIP 曲也能拿到完整歌曲，远端音源 ?refresh=1 即时热换、实例空闲自动重装；并做过实测级性能治理与一轮全量代码审查——二级面板 GPU 开销从 73% 压回底噪，构建产物只部署现役字体。"
image: "images/zlion-home.jpg"
status: "published"
tags:
  - Vue3
  - Vite
  - 个人主页
  - 性能优化
  - 洛雪音源
  - serverless
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/Zlion-Y/homepage"
  - label: "在线访问"
    icon: "material-symbols:home"
    value: "https://www.zlion.top"
lang: "zh_CN"
---

::github{repo="Zlion-Y/homepage"}

参考 [imsyy/home](https://github.com/imsyy/home)（MIT License，已存档）重写的个人导航主页：**唯一运行时依赖就是 vue 本身**，构建产物 gzip 约 60KB（JS 52KB + CSS 9KB），零环境变量部署到 Vercel。

## 特性

- **Bento 网格布局**：一言、时钟、天气长卡、网站导航小卡自由拼装，移动端自动降级单列；主页卡片在 `config.js` 里逐张开关
- **二级「探索更多」面板**：点左上角 Logo 进入，桌面一屏六卡——每日新闻、多平台热榜、音乐播放器、Epic 限免、历史上的今天、站点监控；卡片清单与排列完全由配置数组驱动，删一项即隐藏
- **自愈音乐播放器**：网易云歌单多源并发竞速拉取、音频探针并行选源、断链自动跳下一首，歌词同步居中、点词跳转进度
- **接上洛雪音源**：自定义音源脚本跑在 serverless 函数里（Node 原生 `vm`，一个垫片都不用），多音源按成功率分波对冲、赢家一出即掐断其余、直链探活、连续失败熔断；解析到的直链优先播放，全部失效才回落 Meting——公共 Meting 对 VIP 曲只给 30 秒试听片段，这是它解决的核心问题
- **音源热换与自愈**：`SOURCE_URLS` 指向的远端脚本改完后，请求一次 `/api/health?refresh=1` 即可让在跑的函数实例立刻重装全部音源，不用重新部署；实例空闲超过阈值还会自动整批重装，回收脚本可能遗留的定时器
- **免 Key 天气**：uapis 聚合接口，按访客 IP 定位到县级、IPv6 正常，含 AQI 与多日高低温平滑曲线
- **站点监控**：`fetch(no-cors)` 由访客浏览器直连探测各站点连通性与响应耗时，绿红点实时显示，无需任何第三方监控服务
- **毛玻璃质感与动效**：backdrop-filter 渐变卡片、FluentPlayer 同款「移入才倾斜」的 3D 卡片动效、自定义圆点光标与点击涟漪
- **实测级性能治理**：用 `nvidia-smi` 逐层量化过每一项视觉开销——二级面板那层覆盖整屏的 `backdrop-filter` 因为采样的是"持续在变的主页"，把 GPU 从 49% 推到 73%，改成「不透明壁纸 + 只模糊卡片」后压回 38%（与空白页底噪齐平）；410 行歌单改为渐进上屏后，入场约 100ms 的主线程长任务归零
- **构建即裁剪**：Vite 插件在打包后剔除未启用的 6 款站名字体，部署产物减重约 600KB（开发时 7 款全保留，`config.js` 一键切换照旧）

## 部署

静态站 + 一个 serverless 函数（音源解析）：

```shell
npm install
npm run dev      # 本地开发
npm run build    # 构建产物在 dist/
```

Vercel 上 Framework 自动识别 Vite，`api/url.mjs` + `lib/*` 按根目录 `api/` 约定打包成函数，区域选 `hkg1`（离国内接口最近）；前端调**同源** `/api/url`，所以没有跨域、没有第二个域名和证书要管。

音源脚本放 `sources/` 一起部署；或用 `SOURCE_URLS` 环境变量指向在线脚本——改了远端脚本后不用重新部署，请求一次 `/api/health?refresh=1` 就能让函数立刻重装全部音源，`/api/health` 本身还能看到装上了哪些音源、各自的平台与失败原因。详见仓库 `sources/README.md`。

功能细节、音乐音源的接入方式与部署配置见博客文章[《重写了一份个人主页：Vue 3 + Vite 毛玻璃 Bento，还接上了洛雪音源》](/posts/zlion-home-vue3/)。
