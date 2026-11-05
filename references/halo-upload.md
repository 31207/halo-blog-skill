# Halo 草稿上传

## 前提

- Halo 2.x。
- Node.js >= 22。
- 官方 CLI：`npm install -g @halo-dev/cli`。
- 用户已完成一次登录；Skill 不接触 token。
- Halo 2.20 起 Basic Auth 默认关闭。即使账号密码正确，Basic Auth 也会认证失败；使用 PAT + bearer。若确实需要 Basic Auth，必须在 Halo 启动参数中设置 `halo.security.basic-auth.disabled=false` 并重启。

```bash
halo auth login \
  --profile personal \
  --url https://blog.example.com \
  --auth-type bearer
```

不传 `--token`，让 CLI 通过隐藏输入读取 PAT。凭据保存在系统 keyring。

检查当前配置：

```bash
halo auth current --json
```

如果提示没有 active profile，停止上传，让用户先登录。

可选：读取已有标签和分类，优先复用准确显示名。

```bash
halo post tag list --json --size 100
halo post category list --json --size 100
```

## 上传步骤

### 1. 生成临时 Markdown

路径示例：`/tmp/opencode/halo-blog/<slug>.md`。

```markdown
---
title: "文章标题"
slug: "article-slug"
excerpt: "1–2 句摘要"
tags:
  - "Tag A"
categories:
  - "Category A"
halo:
  publish: false
---

正文……
```

要求：

- `slug` 必须是 ASCII kebab-case。
- 标签/分类写显示名；CLI 会复用已有项，不存在时创建。
- 新建文章不要写 `halo.name` 或 `halo.site`。
- `halo.publish: false` 明确表示草稿。
- 所有字符串建议加引号，避免 YAML 特殊字符问题。

### 2. 上传

```bash
halo post import-markdown \
  --file /tmp/opencode/halo-blog/article-slug.md \
  --json
```

不要加 `--force`，不要加 `--publish`。

### 3. 解析结果

`--json` 输出结构：

```json
{
  "post": {
    "metadata": { "name": "..." },
    "spec": { "publish": false },
    "status": { "permalink": "..." }
  },
  "content": { "...": "..." }
}
```

- 文章名：`post.metadata.name`
- 草稿状态：`post.spec.publish`
- 前台链接：`post.status.permalink`，草稿通常没有

基础地址：

```bash
halo auth current --json
```

取 `baseUrl`。

### 4. 构造链接

```text
编辑：<baseUrl>/console/posts/editor?name=<name>
预览：<baseUrl>/preview/posts/<name>
```

预览需要登录 Halo。

### 5. 验证草稿

```bash
halo post get <name> --json
```

确认 `post.spec.publish` 为 `false`。

### 6. 清理

删除临时文件。用户要求留档时保存到指定路径；CLI 会在导入后改写文件并加入 `halo.name`/`halo.site` 跟踪信息，后续再次导入会更新同一篇文章。

## 草稿保证

- `halo.publish: false`
- 不传 `--publish`
- 不传 `--force`
- 不调用 `PUT /apis/api.console.halo.run/v1alpha1/posts/{name}/publish`
- 上传后重新读取文章确认 `spec.publish` 为 `false`

## 直接 REST API 契约

如果不用 CLI，可以调用：

```http
POST /apis/api.console.halo.run/v1alpha1/posts
Authorization: Bearer pat_...
Content-Type: application/json
```

```json
{
  "post": {
    "apiVersion": "content.halo.run/v1alpha1",
    "kind": "Post",
    "metadata": { "generateName": "post-" },
    "spec": {
      "title": "文章标题",
      "slug": "article-slug",
      "deleted": false,
      "publish": false,
      "pinned": false,
      "allowComment": true,
      "visible": "PUBLIC",
      "priority": 0,
      "excerpt": { "autoGenerate": false, "raw": "摘要" },
      "categories": [],
      "tags": [],
      "htmlMetas": []
    }
  },
  "content": {
    "raw": "Markdown 原文",
    "content": "<p>渲染后的 HTML</p>",
    "rawType": "markdown"
  }
}
```

关键点：

- 不要发送 `headSnapshot`、`baseSnapshot`、`releaseSnapshot`、`owner`。
- `content.content` 必须是 HTML；Halo 不会在这个接口里渲染 Markdown。
- `spec.tags`/`spec.categories` 是 `metadata.name`，不是显示名。
- 标签/分类查询：`GET /apis/content.halo.run/v1alpha1/tags`、`/categories`。
- 创建标签：`POST /apis/content.halo.run/v1alpha1/tags`，`metadata.generateName: "tag-"`。
- 创建分类：`POST /apis/content.halo.run/v1alpha1/categories`，`metadata.generateName: "category-"`。
- 发布：`PUT /apis/api.console.halo.run/v1alpha1/posts/{name}/publish`，默认流程不使用。

## 错误处理

| 现象 | 处理 |
| --- | --- |
| `halo: command not found` | 停止，提示 `npm install -g @halo-dev/cli` |
| `No active Halo profile found` | 让用户执行 `halo auth login` |
| `Authentication failed` / 401 | PAT 无效或过期，重新登录；如果用的是 Basic Auth，Halo 2.20+ 默认已禁用，改用 bearer PAT |
| 403 | 检查 PAT 权限 |
| YAML 解析错误 | 修正引号/缩进 |
| 网络超时 | 先用 `halo post list --keyword <标题> --json` 检查是否已创建 |
| 标签/分类创建失败 | 报告名称，不要继续 |
| `spec.publish` 为 true | 停止并告知用户 |
