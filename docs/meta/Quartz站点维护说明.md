# Quartz 站点维护说明

[在 My Project 中编辑](https://chatgpt.com/space/page_f69d8965a4108191a3aa13d9babc6a09)

> **本文件的性质**：面向编辑者与智能体的 **md → Quartz 发布链操作手册**。
> **不涉及**内容怎么写——那是 [文档维护规则](%E6%96%87%E6%A1%A3%E7%BB%B4%E6%8A%A4%E8%A7%84%E5%88%99.md) 与 [维护说明](%E7%BB%B4%E6%8A%A4%E8%AF%B4%E6%98%8E.md) 的职责；本文件只管"内容写好之后，怎么让它出现在站点上"。
> **适用对象**：一切对 `wiki/` 的新增、修改、删除，以及随之而来的站点重建。
> **用法**：日常只需读[第二节](Quartz%E7%AB%99%E7%82%B9%E7%BB%B4%E6%8A%A4%E8%AF%B4%E6%98%8E.md)（章节：二日常发布三步）；出问题跳[第七节](Quartz%E7%AB%99%E7%82%B9%E7%BB%B4%E6%8A%A4%E8%AF%B4%E6%98%8E.md)（章节：七常见问题排查）。

---

## 一、三层结构与唯一真相源

```
raw/                          metree / notion / vvdworld 导出，只读归档
  │  ← 按《文档维护规则》整理
  ▼
wiki/                    ★★★  唯一真相源。所有编辑都在这里发生
  │  ← migrate-wiki.ps1 搬运（单向、幂等）
  ▼
Quartz/content/                 派生物，可随时整个删除重建
  │  ← npx quartz build
  ▼
Quartz/public/                  派生物，静态站点产物
```

### 三条铁律

| 铁律 | 原因 |
|---|---|
| **① 只编辑 `wiki/`，永不编辑 `Quartz/content/`** | `content/` 是快照。下次迁移会**无条件覆盖**你的改动，白改。 |
| **② `content/` 与 `public/` 不要提交进 git** | 它们由脚本生成，纳入版本控制只会制造无意义的 diff 噪音。 |
| **③ 导航框分类只改 `navbox.groups.yaml`** | 全站导航框共用这一份定义；任何页面都不持有导航框副本。 |

### 各目录职责速查

| 路径 | 是什么 | 能不能手改 |
|---|---|---|
| `wiki/` | 条目正文、模板、目录说明 | ✅ **只能改这里** |
| `meta/` | 编辑者元信息（含本文件） | ✅ 可改（会被发布到 `/meta/`） |
| `Quartz/navbox.groups.yaml` | 导航框分类定义 | ✅ 可改（**导航框的唯一调参处**） |
| `Quartz/bases/*.base` | 数据驱动导航页的查询定义 | ✅ 可改 |
| `Quartz/migrate-wiki.ps1` | 迁移脚本 | ✅ 可改（注意：**必须保持纯 ASCII**，见第七节） |
| `Quartz/quartz.config.yaml` | 站点配置 | ✅ 可改 |
| `Quartz/plugins/navbox/` | 导航框插件 | ✅ 可改（`dist/navbox.js` 是实际加载的文件） |
| `Quartz/quartz/util/glob.ts` | **已打补丁**：移除上游的 `gitignore: true` | ⚠️ **升级 Quartz 会覆盖，见[第七节](Quartz%E7%AB%99%E7%82%B9%E7%BB%B4%E6%8A%A4%E8%AF%B4%E6%98%8E.md)（章节： 症状整站 404但静态资源cssjs还在）** |
| `Quartz/content/` | 迁移产物 | ❌ **禁止手改，会被覆盖** |
| `Quartz/public/` | 构建产物 | ❌ 禁止手改 |
| `Quartz/quartz/` | Quartz 框架源码 | ❌ 别改，升级会冲突 |

---

## 二、日常发布三步

**环境要求**：Node.js ≥ 22（当前实测 v24.11.1）、npm ≥ 10.9.2、Quartz 5.0.0。工作目录 `Mywiki/Quartz/`。

### 第 1 步：在 `wiki/` 里改内容

按 [文档维护规则](%E6%96%87%E6%A1%A3%E7%BB%B4%E6%8A%A4%E8%A7%84%E5%88%99.md) 写条目。**发布链需要你遵守的唯一额外约束**是 frontmatter：

```yaml
---
title: "规范名"          # 必须与文件名一致（去标题性标点）
aliases: ["别名A"]        # 可选；会自动生成别名跳转页
type: 角色                # 决定它出现在哪个 .base 查询里
tags: ["柏格宁线"]         # ★ 决定它出现在哪些导航框里
status: 草稿              # 只能取：完整/较完整/草稿/骨架
canon: 存疑               # 只能取：正典/存疑/旧稿/弃稿
---
```

> `tags` 是导航框的**唯一入场券**。新条目忘了写 tag，它就不会出现在任何导航框里——这不算 bug，是设计。

### 第 2 步：迁移到 `content/`

```powershell
cd D:\myspace\myspace\ocs\Mywiki\Quartz
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
.\migrate-wiki.ps1 -Clean
```

`-Clean` 会先清空 `content/` 再全量重建，**推荐日常都用它**（避免删掉的条目在站点上残留）。不加 `-Clean` 则只做覆盖式更新。

预期输出：

```text
cleaned content/
copied prose files : 62        ← 正文条目数
skipped meta files : 21        ← _templates/_assets 等，正常跳过
published _* nav pages : 11    ← _目录说明.md（正文有链接指向它们）
generated category index.md : 10
injected .base query files : 4
copied meta files : 14
generated site landing page content/index.md
remapped repo-relative links in 1 meta files
normalised ../../meta/ links in 9 files
migration done.
```

**这些数字应当随你的编辑合理变动**。若 `copied prose files` 突然大幅减少，说明源目录出了问题，别继续构建。

### 第 3 步：构建并起服务

```powershell
npx quartz build --serve --port 8080
```

打开 **`http://localhost:8080`** 审阅。改内容后 `--serve` 会自动重建（看到 `Detected change, rebuilding...` 即为正常）。

---

## 三、Quartz 启动命令（完整版）

### 日常开发（推荐）

```powershell
cd D:\myspace\myspace\ocs\Mywiki\Quartz
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
.\migrate-wiki.ps1 -Clean      # 同步 wiki/ → content/
npx quartz build --serve --port 8080
```

访问 `http://localhost:8080` ｜ 退出：`Ctrl + C`

### 只构建，不起服务

```powershell
npx quartz build
```

产物在 `public/`，可直接用任意静态服务器托管。

### 换端口（8080 被占用时）

```powershell
npx quartz build --serve --port 8081
```

### 链接体检

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
.\check-links.ps1
```

**每次发布前跑一次**。有断链会打印明细并以 `exit 1` 结束（可接 CI）。

### 从零重建（换机器 / 出大问题时）

```powershell
cd D:\myspace\myspace\ocs\Mywiki\Quartz
npm install
.\migrate-wiki.ps1 -Clean
npx quartz build --serve --port 8080
```

### 关于 `Set-ExecutionPolicy`

Windows PowerShell 默认禁止运行未签名脚本，不设这一行会报"未对文件进行数字签名"。
`-Scope Process` 只影响当前窗口，**不修改系统设置**。**每次新开 PowerShell 窗口都要重设一次**——
这是最容易忘、也最容易误以为"脚本坏了"的一条。

---

## 四、按场景操作

### 场景 A：修改一个已有条目

```powershell
# 1. 编辑 wiki/04-人物角色/观.md
# 2. 同步
.\migrate-wiki.ps1 -Clean
# 3. 若服务已在跑，它会自动重建；否则
npx quartz build --serve --port 8080
```

### 场景 B：新增一个条目

1. 从 `wiki/_templates/` 复制对应模板到目标分类目录，文件名 = 规范名

2. 填 frontmatter（**特别是 `tags`**）与正文

3. **同步总索引**：`wiki/00-导览与索引/总索引.md` 对应分类下加一行

> ⚠️ **总索引目前仍是手工维护的**。导航框已自动化，但总索引还没有。忘了加，站点上就查不到——这是当前发布链里**唯一还靠人工纪律的一环**。

4. 跑迁移 + 构建（同场景 A）

### 场景 C：删除一个条目

1. 删除 `wiki/` 下的文件

2. **同步总索引**：删掉对应行

3. **必须带 `-Clean` 跑迁移**，否则 `public/` 里会残留旧页面：

```powershell
.\migrate-wiki.ps1 -Clean
npx quartz build
```

### 场景 D：批量同步 raw/ 新内容

这是**内容工作**，规则见 [文档维护规则](%E6%96%87%E6%A1%A3%E7%BB%B4%E6%8A%A4%E8%A7%84%E5%88%99.md) 第二节（raw 内容一律不标注「照抄」）。
发布侧只需注意：新增条目数会体现在 `copied prose files` 上，构建后跑一次 `check-links.ps1` 确认没有指向未建条目的死链。

### 场景 E：调整导航框分组

**只改 `Quartz/navbox.groups.yaml` 一个文件**，然后 `npx quartz build`。全站同步生效，无需碰任何条目页。

```yaml
- title: 故事线                    # 顶层导航框标题
  tags: []
  children:
    - title: 柏格宁线               # 分组名
      tags: [柏格宁线]              # 哪些页面进入本组（按 frontmatter tags 匹配）
      children:                    # 可选：三级
        - title: 角色
          types: [角色]             # 在父行名单内按 type 再切一刀
```

- **自选组**（有 `tags` / `entries`）：自己挑条目

- **收窄组**（只有 `types`）：在父行已选出的名单里按类型切一刀，不新增、不抢条目

- `exclude: [某条目]`：从本组剔除

- `entries: [某条目]`：手工补入（写 `title`，不是文件名）

### 场景 F：新增数据驱动导航页（Bases 查询）

1. 在 `Quartz/bases/` 新建 `xxx.base`（YAML）

2. 跑迁移，它会自动注入到 `content/00-导览与索引/` 并在首页列表中登记

> `bases/` 是**源目录**，`content/` 里的副本是派生物——**改 `.base` 要改 `bases/` 下的那份**，否则 `-Clean` 会冲掉。

---

## 五、发布前检查清单

```powershell
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force
cd D:\myspace\myspace\ocs\Mywiki\Quartz
.\migrate-wiki.ps1 -Clean
npx quartz build
.\check-links.ps1
```

逐项对照：

- [ ] `migrate-wiki.ps1` 输出的正文条目数符合预期（无异常骤降）

- [ ] 构建无 `ERROR` / `Failed to emit` / `failed to load`

- [ ] `check-links.ps1` 报 **broken: 0**

- [ ] 新条目在总索引里有登记

- [ ] 新条目的 `tags` 已填，且能在导航框里看到自己

- [ ] `status` / `canon` 取值未越界（只取枚举值，**不要写括号说明**）

### 当前基线（2026-10-02，供对照）

| 指标 | 数值 |
|---|---|
| `wiki/` md 总数 | 80 |
| 迁移输出 | 正文 60 / 导览页 10 / 分类页 10 / .base 4 / meta 15 |
| 构建输入 → 输出 | 96 → 263 文件 |
| HTML 页数 | 211 |
| 站内链接 | 9669 |
| **断链** | **0** |

> 这些数字会随编辑增长。**关键是 `Found` 非零、构建无 `ERROR`、`broken: 0`**，而不是具体数值。

---

## 六、为什么要有"迁移"这一步

**问题**：`wiki/` 与 Quartz 的 `content/` 目录要求不同——

| 差异 | `wiki/`（人的工作区） | `content/`（Quartz 要求） |
|---|---|---|
| `_目录说明.md` | 每目录一份，编辑者看 | `_` 开头会被构建忽略，但正文有链接指向它 |
| `meta/` | 在 `wiki/` **之外** | 需发布到 `/meta/`，否则链接全断 |
| 相对链接深度 | 从 `Mywiki/` 根算（`../../meta/x.md`） | 从 `content/` 根算（`../meta/x.md`） |
| 分类导航 | 无 | 每目录需 `index.md` |

**`migrate-wiki.ps1` 就是消化这些差异的适配层。** 它做的事：

| 步骤 | 说明 |
|---|---|
| 复制正文 | `wiki/**` 去掉 `_templates/`、`_assets/`（62 个） |
| 发布导览页 | `_目录说明.md` / `_导读说明.md` 补标题后发布（11 个） |
| 生成分类页 | 每个 `NN-分类/` 生成 `index.md`，标题取自 `_目录说明.md` |
| 注入 `.base` | `bases/*.base` → 导航目录（4 个） |
| 发布 `meta/` | `meta/` → `content/meta/`（14 个） |
| 链接归一 | `../../meta/` → `../meta/`（补齐层级差） |
| 仓库链接重映射 | `../wiki/NN-xxx/` → `../NN-xxx/` |
| 生成首页 | `content/index.md`，分类表由代码生成 |

**它故意不做的事**：不改写链接风格（不把标准 markdown 链接转成 `[[wikilink]]`，保持源仓库可移植）、不修改正文与 frontmatter、**不修复数据问题**（`status` 越界值原样保留，让它显眼地暴露在"索引-条目状态"表里）。

---

## 七、常见问题排查

### ⚠️ 症状：整站 404，但静态资源（CSS/JS）还在

**这是最危险的一种故障：构建"成功"、退出码为 0，站点却是空的。**

具体表现：

- 浏览器整站 404，连首页都是 404

- `public/` 里 CSS/JS/字体都在，但**没有任何页面 HTML**（只剩 `404.html`）

- 构建日志出现 **`Found 0 input files from content`**

#### 根因（2026-10-02 实际发生并已修复）

Quartz 上游在 `quartz/util/glob.ts` 里给 globby 传了 `gitignore: true`：

```ts
await globby(pattern, { cwd, ignore: ignorePatterns, gitignore: true })
```

`globby` 会**向上找到 git 仓库根，把 `.gitignore` 规则重写到那个根上**，再按 `cwd` 匹配。
本项目结构恰好踩中它的软肋：

```text
Mywiki/                  ← git 仓库根，.gitignore 在这里
  .gitignore             ← 含 Quartz/content/ 等规则
  Quartz/
    content/             ← 构建时 cwd 指这里（仓库的【子目录】）
```

基准错位后**每条规则都退化成"忽略一切"**，输入文件数变成 0。实测对照：

```text
globby("**/*.*", { cwd: "content", gitignore: true })   →   0   ← 坏
globby("**/*.*", { cwd: "content" })                    → 100   ← 好
```

#### 已做的修复

`quartz/util/glob.ts` 中**移除了 `gitignore: true`**，并留有注释说明这是对上游的有意偏离。
`ignorePatterns`（Quartz 自己的排除机制）不受影响，仍在正常工作。

> ⚠️ **升级 Quartz 会覆盖这个修复。** 升级后请检查 `quartz/util/glob.ts`：
> 只要看到 `gitignore: true` 就删掉，否则整站会再次变空。

#### 快速自检

```powershell
npx quartz build 2>&1 | Select-String 'Found '
```

只要出现 `Found 0 input files`，就是这个（或同类）问题——**不要去查页面代码或插件**。

---

### 构建报 `Plugin "navbox" declares components but failed to load them`

插件解析失败。先做语法检查——**它会给出真实报错，而 Quartz 只打印这一行，不显示原因**：

```powershell
node --check plugins\navbox\dist\navbox.js
```

最常见的两个原因：

1. **注释里出现了反引号**。`dist/navbox.js` 里代码与模板字符串混排，注释中一个多余的反引号就会提前闭合字符串，整个模块解析失败。**改这个文件时不要在 `//` 注释里用反引号。**

2. 手工编辑时漏了字段。`renderGroup` 依赖 `tags` / `types` / `exclude` / `entries` 四个数组都存在，手工构造对象时要补全。

### 报 `无法加载文件 ... 未对文件进行数字签名`

忘了设执行策略。见[第三节](Quartz%E7%AB%99%E7%82%B9%E7%BB%B4%E6%8A%A4%E8%AF%B4%E6%98%8E.md)（章节：关于 set executionpolicy）。

### 改了 `.ps1` 脚本后报奇怪的语法错误 / 中文变乱码

`migrate-wiki.ps1` 与 `check-links.ps1` **必须保持纯 ASCII**。Windows PowerShell 5.1 按 ANSI 读取无 BOM 的 UTF-8 文件，脚本里的中文字面量会被读成乱码，进而引发莫名其妙的语法错误。
**中文内容一律从源文件读取或由代码点拼装**，不要直接写在脚本里。

### 站点上出现了已删除的条目

`migrate-wiki.ps1` 没带 `-Clean`。重新用 `-Clean` 跑一次。

### 链接体检报出一批 `../../xxx`

源文件里写了超出发布层级的相对路径。检查该链接在 `wiki/` 里的写法：如果它指向 `Mywiki/` 之外的层级，发布后必然失效。
已知的**仓库级链接**（`../wiki/_templates/...`）已在迁移时降级为行内代码，不再产生死链。

### 导航框里看不到某个新条目

按顺序查：

1. 该条目的 `tags` 是否填了？填的值是否与 `navbox.groups.yaml` 里某个分组的 `tags` **完全一致**（含全角/半角）？

2. 该条目是否被某组的 `exclude` 排除了？

3. 该条目的 `type` 是否在 `TYPE_ORDER` 里？（未知类型会排到最后，但不会消失）

4. 是否重跑了 `npx quartz build`？（导航框是构建期物化的）

### 端口 8080 被占用

```powershell
npx quartz build --serve --port 8081
```

---

## 八、命令速查

```powershell
# ── 一次性环境准备（每个新窗口）─────────────────────────
cd D:\myspace\myspace\ocs\Mywiki\Quartz
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass -Force

# ── 发布（日常）────────────────────────────────────────
.\migrate-wiki.ps1 -Clean          # wiki/ → content/（全量重建）
npx quartz build --serve --port 8080   # 构建 + 起服务
# 打开 http://localhost:8080  →  Ctrl+C 退出

# ── 只构建 ────────────────────────────────────────────
npx quartz build

# ── 验收 ──────────────────────────────────────────────
.\check-links.ps1                  # 断链体检（应输出 broken: 0）

# ── 排障 ──────────────────────────────────────────────
node --check plugins\navbox\dist\navbox.js    # 导航框插件语法检查

# ── 从零重建 ──────────────────────────────────────────
npm install
.\migrate-wiki.ps1 -Clean
npx quartz build --serve --port 8080
```

---

## 九、相关文档

| 文档 | 职责 |
|---|---|
| [文档维护规则](%E6%96%87%E6%A1%A3%E7%BB%B4%E6%8A%A4%E8%A7%84%E5%88%99.md) | **内容怎么写**（编辑者身份与语气、raw 收录方式、命名与元数据、自查清单）——优先级最高 |
| [维护说明](%E7%BB%B4%E6%8A%A4%E8%AF%B4%E6%98%8E.md) | 仓库总览、目录结构、版本记录 |
| [术语规范与别名对照](%E6%9C%AF%E8%AF%AD%E8%A7%84%E8%8C%83%E4%B8%8E%E5%88%AB%E5%90%8D%E5%AF%B9%E7%85%A7.md) | 规范名与别名 |
| [待决问题与冲突清单](%E5%BE%85%E5%86%B3%E9%97%AE%E9%A2%98%E4%B8%8E%E5%86%B2%E7%AA%81%E6%B8%85%E5%8D%95.md) | 冲突台账 |
| **本文件** | **md → Quartz 发布链：迁移、构建、起服务、排障** |
| `Quartz/README-DEPLOY.md` | 部署细节、站点配置说明、已知限制 |

---

## 十、已知限制

1. **总索引仍需手工维护**：`wiki/00-导览与索引/总索引.md` 不参与自动化。新条目必须手工登记（见场景 B）。

2. **导航框是构建期物化的静态 HTML**：不在浏览器端实时更新，改完必须重新构建。

3. **`content/` 是快照不是活链接**：改 `wiki/` 后必须重跑迁移，站点才会跟上。

4. **`created-modified-date` 拿不到真实 git 时间**：`content/` 未纳入 git，只能用文件 mtime。

5. **无协作/权限/评论**：Quartz 是静态生成器，不是协作 wiki。

6. **本地预览用 `localhost:8080`**：要发布到线上需改 `quartz.config.yaml` 的 `baseUrl`，并配置托管（GitHub Pages / Cloudflare / Netlify）。

---

*本文件由发布链的实际实现过程整理而成 ｜ 环境：Node v24.11.1 + Quartz 5.0.0 ｜ 2026-10-01*

# 页面关联

## 本页引用

- [待决问题与冲突清单](%E5%BE%85%E5%86%B3%E9%97%AE%E9%A2%98%E4%B8%8E%E5%86%B2%E7%AA%81%E6%B8%85%E5%8D%95.md)

- [文档维护规则](%E6%96%87%E6%A1%A3%E7%BB%B4%E6%8A%A4%E8%A7%84%E5%88%99.md)

- [术语规范与别名对照](%E6%9C%AF%E8%AF%AD%E8%A7%84%E8%8C%83%E4%B8%8E%E5%88%AB%E5%90%8D%E5%AF%B9%E7%85%A7.md)

- [世界观企划 Wiki](%E7%BB%B4%E6%8A%A4%E8%AF%B4%E6%98%8E.md)

## 引用本页

- [世界观企划 Wiki](%E7%BB%B4%E6%8A%A4%E8%AF%B4%E6%98%8E.md)
