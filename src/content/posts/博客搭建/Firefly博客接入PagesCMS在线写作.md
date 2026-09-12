---
title: Firefly 博客接入 Pages CMS：浏览器里写文章，不用本地环境
published: 2026-09-12
updated: 2026-09-12
description: 给 Firefly（Astro）博客仓库放一份 .pages.yml，接入 Pages CMS 后在浏览器里就能写文章、传图片、改独立页面，保存即提交 GitHub 并自动部署；附完整配置与踩坑记录
tags: [Firefly, Astro, PagesCMS, 博客搭建]
category: 博客搭建
draft: false
pinned: false
slug: firefly-pages-cms-setup
author: 世琰
---

> 本文档面向使用 Firefly（Astro）主题、仓库托管在 GitHub 的博主，涵盖 Pages CMS 的接入步骤、`.pages.yml` 逐段讲解与踩坑记录。
> **适用主题：** Firefly（Astro 内容集合架构，其他 Astro 博客同样适用）
> **适用平台：** [app.pagescms.org](https://app.pagescms.org)（浏览器内操作，无需本地环境）

---

## 一、Pages CMS 是什么

[Pages CMS](https://pagescms.org) 是一个架在 GitHub 仓库之上的在线内容管理后台：登录后选好仓库和分支，它把仓库里的 Markdown 文章渲染成可视化的编辑界面，写完点保存就是一次 Git 提交。

对静态博客来说，这条链路非常舒服：

```mermaid
graph LR
    A[浏览器里写作] --> B[Pages CMS 保存]
    B --> C[GitHub 自动 commit]
    C --> D[部署平台构建]
    D --> E[网站更新]
```

不用开电脑上的编辑器，不用敲 git 命令，手机、平板、别人的电脑都能写；部署平台（Vercel / Cloudflare Pages / EdgeOne 等）本来就会在收到提交后自动构建，所以「保存」约等于「发布」。

> [!TIP] 建议
> 如果你在本地已经有一套顺手的写作流程，Pages CMS 也不冲突：出门在外用 CMS 补一篇，回来 git pull 接着写就行，两边操作的是同一个仓库。

---

## 二、接入三步走

### 2.1 仓库根目录新建 `.pages.yml`

这是 Pages CMS 的唯一配置文件，它读取内容集合、字段、媒体目录全靠它。一份能直接用的完整配置在[第四节](#四完整配置可直接抄)，抄进去先跑通，再看第五节的逐段讲解按需调整。

### 2.2 建好媒体上传目录

配置里会把上传图片的目录指到 `public/images`，空目录 Git 不会跟踪，先放一个占位文件：

```bash
mkdir -p public/images

touch public/images/.gitkeep
```

### 2.3 打开 CMS 确认

登录 [app.pagescms.org](https://app.pagescms.org)（用 GitHub 账号授权），选中你的仓库和分支，左侧就会出现配置里定义的内容集合和 Media 面板。侧边栏 Admin 里的 **Configuration** 页面显示的就是 `.pages.yml` 本身，以后想改配置也可以直接在这个输入框里改，保存即提交。

> [!NOTE] 提示
> Pages CMS 按分支读配置，确认页面地址里的分支和你部署用的分支一致（Firefly 默认是 `master`）。

---

## 三、避坑前置：字段必须覆盖 front matter

这一节是整篇文章最重要的一句话，先说结论：

> [!WARNING] 警告
> **CMS 编辑器里没有的字段，保存时会被直接从 front matter 里删掉。**

Pages CMS 保存一篇文章时，是按 `.pages.yml` 里定义的字段列表重新生成 front matter 的。我在接入过程中就实际中过招：旧配置里没有定义 `author` 字段，用 CMS 保存一次文章后，front matter 里的 `author: 世琰` 就没了。

所以动手之前，先翻一下自己文章的 front matter，把所有想保留的键都在配置里给出对应字段——包括 `author`、`password`、`series` 这类低频字段。Firefly 主题自带的字段全集见 `src/content.config.ts`，照着抄不会漏。

---

## 四、完整配置（可直接抄）

```yaml
# Pages CMS 配置，字段与 src/content.config.ts 的 Astro 内容集合一一对应；
# 留空的字段保存时会被自动省略，交给 Astro 的 zod 默认值处理，不会写出 null / ""。

media:
  input: public/images        # 上传的文件存到仓库的这个目录
  output: /images             # 写进文章里的 URL 前缀
  rename: random              # 上传时随机重命名，避免中文文件名和重名覆盖

content:
  # ── 文章 ─────────────────────────────────────────────
  - name: posts
    label: 文章
    type: collection
    path: src/content/posts
    format: yaml-frontmatter
    filename:
      template: '{year}-{month}-{day}-{primary}.md'
      field: create           # 新建文章时弹出文件名输入框，自己起名
    view:
      fields: [title, published, draft, category]
      primary: title
      sort: [published]
      default:
        sort: published
        order: desc
    # 集合会把目录里的图片也当成条目列出来，把已有配图的文件名列进来即可
    exclude: [cover.png, hero.png, my-cover.png]
    fields:
      - name: title
        label: 标题
        type: string
        required: true
      - name: published
        label: 发布日期
        type: date
        required: true
      - name: updated
        label: 更新日期
        type: date
        default: ''           # 不自动填今天，需要时再手动选
      - name: draft
        label: 草稿
        type: boolean
        default: false
      - name: description
        label: 摘要
        type: text
      - name: image
        label: 封面图
        type: image
      - name: category
        label: 分类
        type: string
      - name: tags
        label: 标签
        type: string
        list: true
      - name: pinned
        label: 置顶
        type: boolean
        default: false
      - name: lang
        label: 语言
        type: string
        description: 留空跟随站点默认
      - name: author
        label: 作者
        type: string
      - name: comment
        label: 开启评论
        type: boolean
        default: true
      - name: password
        label: 访问密码
        type: string
        description: 留空则不加密
      - name: passwordHint
        label: 密码提示
        type: string
      - name: series
        label: 系列名
        type: string
      - name: seriesOrder
        label: 系列序号
        type: number
      - name: sourceLink
        label: 原文链接
        type: string
      - name: licenseName
        label: 许可证名称
        type: string
      - name: licenseUrl
        label: 许可证链接
        type: string
      - name: body
        label: 正文
        type: rich-text

  # ── 动态（说说 / 微博式短内容）────────────────────────
  - name: dynamic
    label: 动态
    type: collection
    path: src/content/dynamic
    format: yaml-frontmatter
    filename: '{year}-{month}-{day}-{hour}{minute}{second}.md'
    view:
      fields: [published, pinned, location]
      primary: published
      sort: [published]
      default:
        sort: published
        order: desc
    fields:
      - name: published
        label: 发布时间
        type: date
        required: true
        options:
          time: true
          format: 'yyyy-MM-dd HH:mm:ss'
      - name: pinned
        label: 置顶
        type: boolean
        default: false
      - name: location
        label: 位置
        type: string
      - name: body
        label: 内容
        type: rich-text

  # ── 独立页面（about / friends / guestbook）────────────
  # 路由在主题里是写死的，禁用新建 / 重命名 / 删除
  - name: spec
    label: 独立页面
    type: collection
    path: src/content/spec
    format: yaml-frontmatter
    operations:
      create: false
      rename: false
      delete: false
    view:
      fields: [title, description]
      primary: title
    fields:
      - name: title
        label: 页面标题
        type: string
      - name: description
        label: 页面描述
        type: string
      - name: body
        label: 页面内容
        type: text           # 纯文本源码模式，防止富文本弄乱 JSX

  # ── 项目展示（可选，不需要就整段删掉）────────────────
  - name: projects
    label: 项目
    type: collection
    path: src/content/projects
    format: yaml-frontmatter
    filename:
      template: '{primary}.md'
      field: create
    exclude: [images]
    view:
      fields: [title, published, status, order]
      primary: title
      sort: [published]
      default:
        sort: published
        order: desc
    fields:
      - name: title
        label: 项目名
        type: string
        required: true
      - name: published
        label: 发布日期
        type: date
        required: true
      - name: draft
        label: 草稿
        type: boolean
        default: false
      - name: order
        label: 排序权重
        type: number
        description: 越大越靠前
      - name: description
        label: 描述
        type: text
      - name: image
        label: 封面图
        type: image
      - name: status
        label: 状态
        type: select
        options:
          values: [planning, developing, published, archived]
          placeholder: 未设置
      - name: tags
        label: 标签
        type: string
        list: true
      - name: lang
        label: 语言
        type: string
      - name: link
        label: 相关链接
        type: object
        list: true
        fields:
          - name: label
            label: 名称
            type: string
          - name: icon
            label: 图标
            type: string
            description: Iconify 图标名，如 fa7-brands:github
          - name: value
            label: 链接地址
            type: string
      - name: body
        label: 详情
        type: rich-text
```

> [!NOTE] 提示
> 我的仓库里 `exclude` 实际排除了 30 个历史配图文件名，上面为了可读性缩成了示例。把你自己 `src/content/posts` 目录下的图片文件名列全就行。

---

## 五、配置逐段拆解

### 5.1 media：图片传到哪、URL 长什么样

| 键 | 含义 |
|------|------|
| `input` | 上传的文件在仓库里的实际存放路径 |
| `output` | 写进文章 front matter / 正文里的 URL 前缀 |
| `rename` | 上传重命名策略：`random` 随机名 / `safe` 拼接原名的安全字符 / `false` 保留原名 |

Firefly 的封面和正文图片同时支持相对路径、`/` 开头的 public 路径和远程 URL，所以 `input: public/images` + `output: /images` 是最省事的组合：所有图片统一进 public，写进文章的就是 `/images/xxx.png`，本地和线上都能直接访问。

### 5.2 filename：新文件的命名规则

- `{year}` `{month}` `{day}` `{hour}` `{minute}` `{second}`：创建时间，零填充；
- `{primary}`：列表视图的主字段（一般是标题），会做 slug 化处理；
- `field: create`：新建时弹出文件名输入框。

文章的文件名（去掉扩展名）就是 URL 的一部分，所以发布后尽量别再改名。动态这类短内容用纯时间戳命名，完全不用操心。

### 5.3 view：列表怎么排

`fields` 是列表里显示的列，`primary` 是主标题列，`default.sort` + `order` 控制默认排序。文章按发布日期倒序，符合直觉。

### 5.4 exclude：把图片从条目列表里藏起来

Pages CMS 的集合会**递归**列出目录下的所有文件——包括 png。不处理的话，文章列表里会混进一堆「空条目」。`exclude` 按**精确文件名**匹配（不支持通配符），把已有配图的文件名列进去就好；新图片一律走 Media 上传到 `public/images`，别再往内容目录里放。

### 5.5 operations：给危险操作上锁

`create` / `rename` / `delete` 三个开关控制集合的新建、重命名、删除。独立页面的路由在主题里是写死的（`about.astro`、`friends.astro`、`guestbook.astro` 分别读固定文件），误删一个页面就是 404，所以全部禁用。

### 5.6 date 字段的 options

```yaml
- name: published
  type: date
  options:
    time: true                        # 带时间选择器
    format: 'yyyy-MM-dd HH:mm:ss'     # 写入文件的格式（date-fns 语法）
```

`default: ''` 可以阻止日期字段自动填当前时间——`updated` 用得上，需要时再手动选。

---

## 六、踩坑记录

### 6.1 日期必须保持「裸格式」，否则构建直接炸

Firefly 的内容 schema 里 `published` 是 `z.date()`，Astro 解析 front matter 时只把**不带引号**的日期识别成 Date：`published: 2026-09-12` 没问题，`published: "2026-09-12"` 会变成字符串，构建报错。

Pages CMS 的 date 字段默认写裸日期，正常用不会出事。风险点在于**绕过 CMS 手改文件**：在 GitHub 网页上编辑时手一抖给日期加了对引号，下次部署就炸。这是静态博客 + 任何 CMS 组合的共性坑，遇到构建失败先查日期。

### 6.2 空字段不会写出 null，但必填字段别空着

CMS 保存时会自动省略值为空的键，正好符合 Astro schema 的默认值语义，`image`、`updated` 这类可选字段留空即可。但 `title`、`published` 这种 schema 里没有默认值的字段一定要填——配置里给它们标了 `required: true`，保存时 CMS 会先拦一道。

### 6.3 纯中文标题生成不了文件名

`{primary}` 生成文件名时会做 slug 化，纯中文标题 slug 化后是**空字符串**。所以文章的 filename 模板用日期打头、并在新建时弹出输入框让你自己起名（中英文都行，Firefly 对中文文件名的 URL 处理是正常的）。

### 6.4 MDX 文件别用富文本编辑器

`friends.mdx` 这类 MDX 文件里可能有 JSX 注释和组件语法，富文本编辑器（Tiptap）往返一遍很可能弄乱结构。所以配置里独立页面的正文用 `type: text`——一个纯文本框，所见即所得地编辑原始 Markdown，保存时一个字节都不会动。

### 6.5 复杂 Markdown 建议切到 source 模式

文章正文的富文本编辑器右上角可以切换 **editor / source** 两种模式。日常写段落、插图用 editor 方便；写 callout（`> [!NOTE]`）、复杂表格、大量代码块时切到 source 模式，保存的是原始 Markdown，最稳。

### 6.6 「via Pages CMS」的提交会规范化文件格式

用 CMS 保存过的文件会被重新序列化：front matter 的行内数组会变成块列表、表格对齐会被重排、多余的空行会被整理。这些都是合法的等价改写，git diff 大不代表内容变了。但如果两个配置的字段列表不一致，就会有第三节的「字段被删」问题——这就是为什么字段要配全。

---

## 七、日常使用

### 7.1 写一篇新文章

1. 侧边栏 → 文章 → 右上角新建；
2. 弹出文件名输入框，填一个 `2026-09-12-my-post` 风格的名字；
3. 填标题（必填）、发布日期默认今天，标签是一行一个；
4. 正文用富文本编辑器写，需要图片就点工具栏的插图按钮直接上传，图会传到 `public/images` 并插入 `/images/xxx.png`；
5. 保存。此时 GitHub 上多一个 commit，部署平台开始构建，一两分钟后线上可见。

### 7.2 草稿流程

没写完的文章把「草稿」勾上（`draft: true`），主题不会把它渲染进列表和详情页，可以放心保存占位。写完取消勾选再保存即发布。

### 7.3 改独立页面

关于页、友链、留言板都在「独立页面」里，正文是原始 Markdown 源码模式，改完直接保存。新增独立页面需要先在主题里加路由，光建 md 文件没用，所以配置里禁掉了新建。

> [!TIP] 建议
> 本地改完东西想用 CMS 写作之前，记得先 `git pull`，反之 CMS 上写完回来也要先 pull——两边是同一个仓库，别让自己处在冲突状态。

---

## 八、故障排查

| 症状 | 可能原因 | 处理 |
|------|----------|------|
| 保存了但网站没更新 | 部署平台还在构建，或构建失败 | 去部署平台看构建日志 |
| 构建失败，报 `z.date()` / Invalid date | 日期被写成了带引号的字符串 | 检查该文 front matter 的日期字段 |
| 构建失败，报某字段类型错误 | 手改文件时 front matter 格式错了 | 对照 `src/content.config.ts` 检查 |
| Media 面板打不开 / 上传失败 | `public/images` 目录不存在 | 建目录并提交 `.gitkeep` |
| 文章列表里出现图片「条目」 | `exclude` 没覆盖到新图片 | 补 exclude，或把图移去 `public/images` |
| 改了 `.pages.yml` 没生效 | 看错了分支，或页面是旧缓存 | 确认分支一致后刷新页面 |

> [!CAUTION] 注意
> 构建失败时部署平台上的还是上一个成功版本，网站不会白屏，但你的修改也不会生效。修好 front matter 再推一次即可。

---

## 九、结语

整套方案的本质是把「能改仓库文件的 GitHub 账号」变成一个带表单校验和媒体库的写作后台：配置成本只有一份 `.pages.yml`，收益是随时随地可写、格式有 CMS 兜底、提交历史照样干净。

如果你也在用 Firefly，欢迎直接抄第四节的配置；踩到别的坑欢迎来[留言板](/guestbook/)交流。

---

> **文档版本：** 2026-09-12
> **适用主题：** Firefly（Astro 内容集合架构）
> **适用平台：** app.pagescms.org / Vercel / Cloudflare Pages / EdgeOne Pages 等自动部署平台
