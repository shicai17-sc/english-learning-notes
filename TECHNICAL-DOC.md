# 英语学习手册 · 技术架构与维护手册

> 最后更新：2026-09-12
> 本文档记录这个网站的完整技术方案、维护操作方法、现有功能清单和后续优化方向，供后期查看与扩展使用。

---

## 目录

1. [项目总览](#1-项目总览)
2. [整体架构](#2-整体架构)
3. [组件详细说明](#3-组件详细说明)
4. [发布与部署流程](#4-发布与部署流程)
5. [后期维护手册](#5-后期维护手册)
6. [网站功能清单](#6-网站功能清单)
7. [后续优化方向](#7-后续优化方向)
8. [附录：账号与密钥清单](#8-附录账号与密钥清单)

---

## 1. 项目总览

| 项目 | 内容 |
|---|---|
| 网站内容 | 1226 条英语学习笔记（问答式精炼笔记） |
| 前台地址 | https://shicai17-sc.github.io/english-learning-notes/ |
| 后台地址 | https://shicai17-sc.github.io/english-learning-notes/admin.html |
| 访问策略 | 所有人免登录可读；仅管理员可修改 |
| 托管方案 | GitHub Pages（前端）+ Supabase（后端），**零服务器成本** |
| 内容分类 | 8 类：句子翻译 735 / 单词与表达 279 / 语法 101 / 英语杂谈 33 / 学习方法 32 / 语言文化 27 / 口语场景 16 / 考试水平 3 |

**设计目标回顾**：不买服务器、免费优先、所有人免登录访问、管理员改动实时生效且全访客同步。

---

## 2. 整体架构

```
┌─────────────┐        HTTPS         ┌──────────────────┐
│   浏览器     │ ──────────────────▶ │  GitHub Pages     │  静态文件托管
│ (手机/电脑)  │                      │  index.html 前台   │
└─────────────┘                      │  admin.html 后台   │
        │                            └──────────────────┘
        │  Supabase JS SDK (anon key)
        ▼
┌─────────────────────────────────────────────┐
│               Supabase 云后端                 │
│  ┌─────────┐ ┌──────────┐ ┌──────────────┐  │
│  │  PostgreSQL │  Auth 认证 │  Storage 存储 │  │
│  │  notes 表    │  邮箱+密码  │  note-images  │  │
│  │  categories │  管理员账号 │  公开读/仅管理员写│  │
│  └─────────┘ └──────────┘ └──────────────┘  │
│  ┌──────────────────────────────────────┐   │
│  │  Realtime 实时推送（notes/categories） │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

**核心分工**：

- **GitHub Pages = 静态 web 服务器**：只负责把 HTML 文件发给浏览器，不执行任何程序。前台、后台都是纯前端页面（HTML + CSS + JS），数据全靠浏览器直接请求 Supabase。
- **Supabase = 动态后端**：数据库、登录认证、文件存储、实时推送全部由它承担。它是一台"托管式"服务器——不需要自己买机器、装环境、打补丁，全免费额度。
- **为什么这套能零成本**：两台服务都是托管式（平台运营方维护），用户只需使用。

**请求链路示例（访客打开前台）**：

1. 浏览器请求 GitHub Pages → 拿到 index.html
2. 页面里的 JS 用 Supabase SDK 拉取 notes 表 1226 条（PostgREST，按 1000 条分页）
3. 同时拉取 categories 表分类定义
4. 渲染列表；订阅 Realtime，后续任何变更自动刷新

---

## 3. 组件详细说明

### 3.1 GitHub Pages（前端托管）

| 项目 | 说明 |
|---|---|
| GitHub 账号 | shicai17-sc |
| 仓库 | https://github.com/shicai17-sc/english-learning-notes |
| 部署机制 | 仓库 `main` 分支 push 后，Pages 自动构建发布（约 1~3 分钟） |
| 仓库根目录 | 本地 `.../english-learning-notes/publish/`（**git 仓库在此目录**） |
| 页面文件 | `index.html`（前台，约 40KB）、`admin.html`（后台，约 28KB） |

**本地目录结构**（工作区）：

```
english-learning-notes/
├── index.html          # 前台成品（由脚本生成，勿手改）
├── admin.html          # 后台成品（可直接修改）
├── publish/            # ← Git 仓库根，push 到 GitHub 的目录
│   ├── index.html
│   └── admin.html
└── _data/
    ├── gen_html.py     # 前台生成脚本（改完重跑再同步）
    ├── test_cloud.py   # 前台端到端测试（Playwright）
    ├── test_admin.py   # 后台门禁测试
    └── supabase_import.py  # 1226 条历史导入脚本（一次性）
```

> ⚠️ **重要约定**：改前台页面 → 改 `_data/gen_html.py` 重跑 → `cp index.html publish/index.html` → 在 `publish/` 里 `git commit + push`。不要直接手改 `index.html`（会被脚本覆盖）。

### 3.2 Supabase（云后端）

| 项目 | 说明 |
|---|---|
| 项目 ref | `cecjcbvxgojpuynirtbq` |
| 项目 URL | `https://cecjcbvxgojpuynirtbq.supabase.co` |
| 地区 | 东京（ap-northeast-1） |
| 管理后台 | https://supabase.com/dashboard/project/cecjcbvxgojpuynirtbq |

#### 数据库表

**notes 表（笔记，1226 行）**

| 字段 | 类型 | 说明 |
|---|---|---|
| id | int8 PK | 自增主键 |
| q | text | 提问（笔记标题） |
| body | text | 答案（Markdown 原文） |
| cat | text | 分类标识（对应 categories.id） |
| date | text | 日期 |
| fups | jsonb | 追问数组 `[{q, a}]` |
| dup | jsonb | 同题重问记录 |
| images | jsonb | 图片 URL 数组，默认 `[]` |
| deleted | bool | 软删除标记（true = 前台隐藏） |
| created_at | timestamptz | 创建时间 |

**categories 表（分类，8 行）**

| 字段 | 类型 | 说明 |
|---|---|---|
| id | text PK | 英文标识（如 `words`，笔记的 cat 字段引用它） |
| name | text | 显示名称（可在后台改） |
| color | text | 分类颜色（hex，可在后台改） |
| sort | int | 排序 |
| description | text | 分类说明 |

#### 权限策略（RLS，安全核心）

所有表启用行级安全（Row Level Security），**即使前端代码被绕过，数据库层面也会拦截非法操作**。

| 表 | 操作 | 允许谁 |
|---|---|---|
| notes | SELECT | 所有人（匿名可读） |
| notes | UPDATE / INSERT / DELETE | 仅 `auth.jwt()->>'email' = 'shicai17@gmail.com'` |
| categories | SELECT | 所有人 |
| categories | UPDATE / INSERT / DELETE | 仅管理员邮箱 |
| storage.objects | SELECT（读文件） | 所有人（桶为 public） |
| storage.objects | INSERT / UPDATE / DELETE | 仅管理员邮箱 |

> **如何增加第二个管理员**：把各策略里的 `'shicai17@gmail.com'` 改成 `in ('shicai17@gmail.com','xxx@gmail.com')`，或新建一个管理员角色。操作方式见 [5.3 权限调整](#53-权限调整)。

#### Auth 认证

- 方式：邮箱 + 密码（Supabase Auth）
- 当前唯一有写权限的账号：`shicai17@gmail.com`（注册后需邮箱确认）
- 前端判断 `isAdmin()` = 登录用户邮箱 === 管理员邮箱；RLS 是真正的防线

#### Storage 存储

- 桶名：`note-images`（公开读）
- 限制：单文件 ≤ 5MB；格式仅 `image/jpeg / image/png / image/webp / image/gif`
- 上传路径规则：`{用户ID}/{时间戳}-{随机}-{文件名}`
- 公开访问 URL 格式：`https://cecjcbvxgojpuynirtbq.supabase.co/storage/v1/object/public/note-images/<路径>`

#### Realtime 实时同步

- 订阅 `notes` 表所有变更 → 400ms 防抖后重新拉取渲染
- 订阅 `categories` 表所有变更 → 重新加载分类并渲染
- 效果：管理员在后台/前台改动，所有打开页面的访客自动看到新内容，无需刷新

### 3.3 前端实现要点

- **单文件自包含**：前台 index.html 一个文件包含全部 CSS/JS，无构建步骤；Supabase SDK 用 jsDelivr CDN 引入（`@supabase/supabase-js@2`）
- **数据分页**：PostgREST 单次最多返回 1000 行，脚本按 `range(from, from+999)` 循环拉全量
- **软删除**：删除 = `deleted=true`（数据保留，可恢复）；恢复 = `deleted=false`；前台查询默认过滤 deleted
- **图片上传**：先传 Storage 拿公开 URL，再连同 `images` 数组一起写入 notes；编辑中取消会清理本次新传的图；移除旧图时同步删除云端文件
- **分类动态化**：分类定义不写死，运行时从 categories 表拉取，后台改名前台即时生效
- **登录弹窗**：邮箱+密码，支持注册（需邮箱确认），登录后管理员按钮才出现

---

## 4. 发布与部署流程

### 4.1 日常改内容（推荐用后台）

打开后台 → 登录 → 编辑/新增/隐藏/传图 → 自动写入数据库 → 前台自动同步。**无需任何部署动作**。

### 4.2 改前台页面代码

```bash
# 1. 修改生成脚本（或 admin.html）
cd /home/user/Doubao/chats/38441430903361794/english-learning-notes/_data
python3 gen_html.py            # 重新生成 index.html

# 2. 同步到部署目录并推送
cp ../index.html ../publish/
cd ../publish
git add -A
git commit -m "说明改了什么"
git push origin main           # Pages 自动发布，1~3 分钟生效

# 3. 验证
curl -s -o /dev/null -w "%{http_code}" https://shicai17-sc.github.io/english-learning-notes/
# 期望 200
```

### 4.3 前端改动自检

```bash
cd _data && python3 test_cloud.py   # 前台 11 项端到端断言
cd _data && python3 test_admin.py   # 后台 5 项门禁断言
```

> 测试用 Playwright 无头浏览器，需指定 chromium：`/opt/vm/preinstall/ms-playwright/chromium_headless_shell-1169/chrome-linux/headless_shell`

---

## 5. 后期维护手册

### 5.1 数据备份

| 方式 | 操作 |
|---|---|
| 页面导出 | 前台侧栏「导出」或后台「导出全部数据」→ JSON 文件 |
| 数据库完整备份 | Supabase Dashboard → Database → Backups（每日自动备份，免费版保留 7 天） |
| 手动全量拉取 | 用 service_role key 请求 `GET https://<ref>.supabase.co/rest/v1/notes?select=*`（可分页） |
| 分类备份 | 同上，表名换成 `categories` |

### 5.2 直接改数据库（SQL）

**推荐入口**：Supabase Dashboard → SQL Editor → New query（图形界面，最安全）。

常用操作示例：

```sql
-- 查看表结构
select column_name, data_type from information_schema.columns
where table_schema = 'public' and table_name = 'notes';

-- 给 notes 加新字段（例如加"来源链接"列）
alter table public.notes add column if not exists source_url text default '';

-- 批量改分类：把某分类全部笔记移到另一分类
update public.notes set cat = 'words' where cat = 'grammar';

-- 批量恢复被隐藏的笔记
update public.notes set deleted = false where deleted = true;

-- 物理删除某条笔记（慎用，不可恢复）
delete from public.notes where id = 123;

-- 改分类名称（等同后台操作）
update public.categories set name = '新名字' where id = 'words';
```

> 所有修改即时生效：前台订阅了 Realtime，打开页面的人会自动看到。

### 5.3 权限调整

```sql
-- 给第二个邮箱加笔记写权限（把现有策略改成多邮箱）
drop policy if exists notes_write on public.notes;
create policy notes_write on public.notes
  for update to authenticated
  using (auth.jwt()->>'email' in ('shicai17@gmail.com', 'second@example.com'));

-- 注意：notes 的 INSERT / DELETE 策略、categories 的写策略、storage.objects 三个策略要同步改
```

### 5.4 恢复被删数据

- 软删除的数据：后台「显示已隐藏」→ 勾选 → 恢复显示
- 物理删除/误操作：用 5.1 的备份恢复（Dashboard Backups 可回滚到前一天）

### 5.5 换 Supabase 项目 / 迁移

1. 新项目建同样的表（可复制 3.2 的建表语句）
2. 导出旧数据 JSON → 导入新项目（可用 `_data/supabase_import.py` 改 URL/密钥复用）
3. 复制 RLS 策略、Storage 桶、Auth 账号
4. 更新前端两个文件里的 `SUPABASE_URL` 和 `SUPABASE_KEY`（anon key），重新部署

### 5.6 免费额度红线（超出会怎样）

| 资源 | 免费额度 | 说明 |
|---|---|---|
| 数据库 | 500MB | 当前数据量约几 MB，远未触及 |
| Storage | 1GB | 图片多了要注意，超了可清旧图 |
| Auth 用户 | 5 万 MAU | 当前仅 1 个真实账号 |
| Realtime 并发 | 200 连接 | 同时在线访客超过会排队 |
| GitHub Pages | 100GB/月流量、1GB 站点大小 | 当前页面共约 68KB，无压力 |

**省钱提示**：本项目永远不需要付费——内容型站点用量远低于免费线。

### 5.7 已知限制

- GitHub Pages 只能托管静态文件：任何"服务端逻辑"（定时任务、邮件、AI 处理）都需走 Supabase Edge Functions 或外部服务
- 图片单张 5MB 上限（前端+后端双限制）
- 前台页面是脚本生成的：手改 index.html 会被下次生成覆盖
- 图片删除是"跟随笔记"的：隐藏笔记不会删图；彻底删图需在编辑表单里移除

---

## 6. 网站功能清单

### 6.1 前台（访客可见）

| 功能 | 说明 |
|---|---|
| 浏览全部笔记 | 1226 条，按分类分组展示 |
| 搜索 | 关键词实时过滤（匹配提问+答案），显示命中数 |
| 分类筛选 | 侧栏按 8 个分类查看 |
| 展开/收起 | 长答案折叠，点「展开全部」查看 |
| 图片查看 | 有图笔记显示缩略图，点击放大、再点关闭 |
| 管理员登录 | 右上角登录弹窗（邮箱+密码+注册） |
| 修改笔记 | 管理员登录后每条的「修改」按钮：改提问/答案/分类、传图/删图 |
| 删除/恢复 | 管理员「删除」= 软删（侧栏可恢复） |
| 导出数据 | 一键下载全部笔记 JSON |
| 实时同步 | 他人改动自动刷新，无需手动刷新页面 |

### 6.2 后台（管理员专用，/admin.html）

| 模块 | 功能 |
|---|---|
| 笔记管理 | 搜索（提问+答案）、按分类筛选、「显示已隐藏」开关；单条编辑/隐藏/恢复；**批量勾选**隐藏/恢复；**新增笔记** |
| 编辑表单 | 提问、分类下拉、答案（Markdown）、图片多选上传/预览/移除 |
| 分类管理 | 每分类改名、改颜色（保存即前台生效）；新增分类（英文标识+名称+颜色）；无笔记的分类可删除 |
| 数据统计 | 总笔记数、显示中、已隐藏、图片总数、分类数、分类分布条 |
| 导出 | 导出全量（含已隐藏）JSON 备份 |
| 权限门禁 | 未登录/非管理员只看到登录框，看不到任何数据 |

### 6.3 权限矩阵

| 操作 | 访客（未登录） | 管理员 |
|---|---|---|
| 浏览/搜索/看图片 | ✅ | ✅ |
| 登录/注册 | ✅ | ✅ |
| 修改/删除/新增/上传 | ❌（按钮隐藏+RLS 拦截） | ✅ |
| 进后台 | ❌ | ✅ |

---

## 7. 后续优化方向

以下方向均基于现有架构（GitHub Pages + Supabase），按"想加什么"给出实现路径，不需要换技术栈。

| 想做的功能 | 实现路径 | 复杂度 |
|---|---|---|
| 访客评论/留言 | 新建 `comments` 表（note_id, content, created_at, RLS 允许匿名 insert 但限制频率）→ 前台渲染评论区 | 中 |
| 访客点赞/收藏 | 新建 `likes` 表（user_id, note_id）按用户隔离，RLS 绑定 auth.uid() | 中 |
| 多管理员/协作者 | 改 RLS 为邮箱列表或建 `admins` 表，后台加成员管理界面 | 低 |
| 定时任务（自动备份、每日统计邮件） | Supabase Edge Functions（Deno）+ pg_cron 定时触发 | 中高 |
| AI 能力（自动生成答案、语音朗读） | Edge Functions 调大模型 API，前端加按钮调用 | 中高 |
| 自定义域名 | 买域名 → GitHub Pages 仓库 Settings 绑定（免费，需自己付费域名） | 低 |
| 独立图片库管理 | Storage 增加 list 策略（仅管理员）→ 后台加"图片管理"tab：查看/删除孤儿图 | 中 |
| 操作日志 | 新建 `audit_log` 表，修改/删除时前端写一条记录，后台可查 | 低 |
| 笔记导入/导出 Excel | 导出走现有 JSON；导入可写脚本解析 CSV 后调 REST 批量 insert | 中 |
| 深色模式 | 前台 CSS 加 `prefers-color-scheme` 变量切换 | 低 |
| 统计访问量 | 免费方案：Supabase 建 `pageviews` 表前端埋点，或接第三方（如 Umami 免费自托管） | 低 |

**优化原则**：优先做"不动现有数据结构"的功能（加新表，不改 notes 主结构）；改动 RLS 时先在 SQL Editor 里用匿名/非管理员账号验证被拦截。

---

## 8. 附录：账号与密钥清单

> ⚠️ **安全提示**：本仓库是**公开仓库**，以下密钥值**不写入此文档**。请记住获取位置，或保存在自己的私密笔记里。

| 项目 | 值/位置 | 敏感性 |
|---|---|---|
| Supabase 项目 ref | `cecjcbvxgojpuynirtbq` | 公开 |
| Supabase 项目 URL | `https://cecjcbvxgojpuynirtbq.supabase.co` | 公开 |
| anon key（前端在用） | 见 index.html/admin.html 内 `SUPABASE_KEY` | 公开（设计上可暴露） |
| **service_role key** | Supabase Dashboard → Settings → API Keys | 🔴 超级权限，**严禁**写进前端/公开仓库 |
| **Personal Access Token** | Supabase Dashboard → Account → Access Tokens | 🔴 可管理整个项目，**严禁**公开 |
| 管理员邮箱 | shicai17@gmail.com | 公开（页面可见） |
| 管理员密码 | 你注册时设置 | 🔴 仅自己知道 |
| GitHub 账号 | shicai17-sc | 公开 |
| GitHub 仓库 | shicai17-sc/english-learning-notes | 公开 |

**密钥使用场景**：

- **anon key**：前端两个 HTML 里写死，任何访客可见，只能读公开数据和登录——安全。
- **service_role key**：脚本/后台管理用（导入数据、清理、跨用户操作）。泄露 = 任何人可读写全库。**只用在你本地命令行或服务端，永远不进页面代码。**
- **PAT**：supabase CLI 登录用（`supabase login`），同样只在本机。

---

*文档结束。修改任何架构后，请同步更新本文档对应章节。*
