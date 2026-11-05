# Halo Blog Skill

把当前开发对话中的技术知识整理成可独立阅读的中文技术文章，并保存为 Halo 草稿。

## 特性

- 从当前会话提取问题、原因、证据、方案和边界，不复制对话。
- 按请求调整文章结构：完整文章、原理分析、排查记录、短笔记、多知识点合并。
- 自动生成标题、摘要、Slug、标签和分类建议。
- 上传前展示全文和元数据，等待明确确认。
- 默认只创建 Halo 草稿，不发布、不更新、不删除。
- 复用 Halo 官方 CLI：Markdown 渲染、标签/分类解析、keyring 凭据存储都由 CLI 处理。

## 前置要求

- OpenCode。
- Halo 2.x。
- Node.js >= 22。
- Halo 官方 CLI：

```bash
npm install -g @halo-dev/cli
```

## 安装

把本目录放到 OpenCode 的 skills 路径之一：

```text
~/.agents/skills/halo-blog/
# 或
~/.config/opencode/skills/halo-blog/
```

重启 OpenCode 后生效。Skill 名为 `halo-blog`。

## Halo 认证

使用 Personal Access Token（PAT），不要使用 Basic Auth。

```bash
halo auth login \
  --profile personal \
  --url https://blog.example.com \
  --auth-type bearer
```

不传 `--token`，按提示输入 PAT。凭据保存在系统 keyring，Skill 不会接触 token。

验证：

```bash
halo auth current --json
```

> Halo 2.20 起 Basic Auth 默认关闭。即使账号密码正确，Basic Auth 也会认证失败。若必须使用，需在 Halo 启动参数中设置 `halo.security.basic-auth.disabled=false` 并重启。

## 使用

在开发对话后直接说：

- 整理成博客
- 把刚才的知识总结一下，上传到 Halo
- 只整理 Rust Pin 相关的内容
- 把这次排查过程整理成技术文章
- 整理成一篇简短的技术笔记

“整理成博客”只是起草请求。Skill 会先展示文章和元数据，只有明确回复“确认上传”才会创建草稿。

## 工作流

1. 锁定范围：只保留指定主题，删除无关内容和试错。
2. 提取事实卡片：问题、原因、证据、方案、边界。
3. 选择结构：完整文章、原理、排查、短笔记或多知识点。
4. 起草：中文写作，技术名词保留英文，代码块标注语言。
5. 自检：事实准确性、独立阅读性、脱敏、元数据。
6. 确认：展示标题、Slug、摘要、标签、分类、字数和全文。
7. 上传：创建 Halo 草稿。
8. 验证：确认 `spec.publish` 为 `false`，返回编辑和预览链接。

## 上传机制

Skill 使用 Halo 官方 CLI 的 `import-markdown`：

```bash
halo post import-markdown \
  --file /tmp/opencode/halo-blog/article.md \
  --json
```

临时 Markdown 的 frontmatter：

```yaml
---
title: "文章标题"
slug: "article-slug"
excerpt: "摘要"
tags:
  - "Tag A"
categories:
  - "Category A"
halo:
  publish: false
---
```

约定：

- `slug` 必须是 ASCII kebab-case，不能依赖 CLI 从中文标题推导。
- 标签和分类写显示名；CLI 会复用已有项，不存在时创建。
- 新建文章不写 `halo.name` 或 `halo.site`，避免误更新已有文章。
- 不传 `--publish`，不传 `--force`。
- 上传后验证 `spec.publish` 为 `false`。

## 安全边界

- 不发布文章。
- 不更新、不删除已有文章。
- 不把 token 写入 Skill、命令、日志或对话。
- 不自动安装 CLI。
- 上传前脱敏：token、密码、私钥、cookie、内部域名、IP、客户名、个人路径。
- 首版不处理图片上传；图片必须是可公开访问的 URL，或在 Halo 编辑器中补充。

## 目录结构

```text
halo-blog/
├── SKILL.md                 # 触发条件、工作流、写作规则、确认门禁
├── README.md                # 人类可读说明
└── references/
    └── halo-upload.md       # CLI 命令、REST API 契约、错误处理
```

## 故障排查

| 现象 | 处理 |
| --- | --- |
| `halo: command not found` | `npm install -g @halo-dev/cli` |
| `No active Halo profile found` | 运行 `halo auth login` |
| Basic Auth 认证失败 | Halo 2.20+ 默认禁用 Basic Auth，改用 bearer PAT |
| 401 | PAT 无效或过期，重新登录 |
| 403 | 检查 PAT 是否有文章、标签、分类权限 |
| YAML 解析错误 | 检查 frontmatter 引号和缩进 |
| 网络超时 | 先用 `halo post list --keyword <标题> --json` 检查是否已创建 |
| `spec.publish` 为 true | 停止操作并告知用户 |

## 相关链接

- [OpenCode Agent Skills](https://opencode.ai/docs/skills)
- [Halo REST API](https://docs.halo.run/developer-guide/restful-api/introduction)
- [Halo CLI](https://github.com/halo-dev/cli)
