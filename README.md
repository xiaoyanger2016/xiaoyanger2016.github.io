# 我的开发笔记

个人开发博客 · Hexo + [Aurora](https://github.com/auroral-ui/hexo-theme-aurora) 主题 · GitHub Pages

线上地址：https://xiaoyanger2016.github.io

## 工作流：AI 起草 → 我审核 → 发布

```
我给要点/草稿  →  AI 扩写成草稿(source/_drafts/)  →  我本地预览+脱敏  →  hexo publish 发布 + push  →  Actions 自动上线
```

> **审核闸门**：草稿放在 `source/_drafts/`，`hexo generate` 默认**不会**渲染草稿，所以没经我审核的内容永远不会上线。只有 `hexo publish` 把它移到 `source/_posts/` 后，push 才会发布。

### 1. 让 AI 写一篇

把一段开发经验的要点丢给 AI，让它生成 `source/_drafts/<slug>.md`。
或手动建：

```bash
hexo new draft "我的标题"      # 生成 source/_drafts/我的标题.md
```

### 2. 本地预览（含草稿）

```bash
hexo server --draft            # -draft 渲染草稿，访问 http://localhost:4000
```

### 3. 脱敏审核（⚠️ 上线前必做）

逐条核对，确认**没有**泄露：

- [ ] 公司/客户/项目内部名称、内部系统名（工单号、内部仓库名、内部域名）
- [ ] 真实的 API key / token / 密码 / 内网 IP / 数据库连接串
- [ ] 未公开的业务逻辑、架构细节、客户数据
- [ ] 真实的人名、邮箱、Slack/聊天截图
- [ ] 代码片段里的内部路径、内部包名

> 经验文章应抽象成**通用技术问题**（如「Laravel 队列重复消费怎么排查」），而不是「我们公司 X 系统在 Y 单里的 bug」。

### 4. 发布

确认无误后，把草稿正式发布并推送：

```bash
hexo publish 我的标题          # source/_drafts/ → source/_posts/
git add -A && git commit -m "post: 我的标题" && git push
```

GitHub Actions 会自动构建并部署，1～2 分钟后上线。

## 本地命令速查

| 操作 | 命令 |
|------|------|
| 预览(含草稿) | `hexo server --draft` |
| 预览(仅正式) | `hexo server` |
| 生产构建测试 | `hexo clean && hexo generate` |
| 新建草稿 | `hexo new draft "标题"` |
| 发布草稿 | `hexo publish "标题"` |

## 技术栈

- **Hexo** + **Aurora** 主题（`hexo-theme-aurora` + `hexo-plugin-aurora`，npm 安装）
- GitHub Pages 托管，GitHub Actions（`.github/workflows/deploy.yml`）自动部署

### 兼容性注意

Aurora（2024 年）的代码高亮器期望 **marked v4 的旧渲染签名**，因此本仓库把 `hexo-renderer-marked` 锁定在 `^6.3.0`（对应 `marked@4`）。**升级 `hexo-renderer-marked` 到 7+（marked 15）会导致构建报 `code.split is not a function`**，请勿升级。

### 本地开发起步

```bash
npm install          # 安装依赖（含锁定版本的 renderer）
hexo server --draft  # 本地预览
```

### 主题配置

- 站点配置：`_config.yml`
- 主题配置：`_config.aurora.yml`（站点信息、菜单、社交、评论、shiki 高亮等）
