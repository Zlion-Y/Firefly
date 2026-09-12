---
title: "zlion-home"
slug: zlion-home
published: 2026-09-12
draft: false
description: "参考 imsyy/home 重写的个人导航主页：Vue 3 + Vite 单依赖实现毛玻璃 Bento 布局、卡片 3D 动效、高德天气与一言，构建产物 gzip 仅约 35KB，GitHub + Vercel 免费部署。"
image: "images/zlion-home.png"
status: "published"
tags:
  - Vue3
  - Vite
  - 个人主页
link:
  - label: "GitHub"
    icon: "fa7-brands:github"
    value: "https://github.com/Zlion-Y/zlion-home"
  - label: "在线访问"
    icon: "material-symbols:home"
    value: "https://www.zlion.top"
lang: "zh_CN"
---

::github{repo="Zlion-Y/zlion-home"}

参考 [imsyy/home](https://github.com/imsyy/home)（MIT License，已存档）重写的个人导航主页，只用 Vue 本身这一个运行时依赖，构建产物 gzip 约 35KB。

## 特性

- **Bento 网格布局**：一言、时钟、天气长卡、网站导航小卡自由拼装，移动端自动降级单列
- **毛玻璃质感**：backdrop-filter 渐变卡片 + FluentPlayer 同款「移入才倾斜」的 3D 卡片动效
- **实用信息聚合**：一言、实时时钟、高德天气（精确到市区，含平滑温度曲线）
- **自定义光标涟漪**：点击波纹动效，桌面端专属
- **极致轻量**：Vue 3.5 + Vite 7，无组件库、无状态管理库，零配置部署到 Vercel

## 部署

```shell
pnpm install
pnpm dev      # 本地开发
pnpm build    # 构建产物在 dist/
```

从选型、布局草图到动效踩坑的完整记录见博客文章[《参考 imsyy/home 重写的个人主页》](/posts/zlion-home-vue3/)。
