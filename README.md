# 官方技术文档，一键查 —— Tech Doc Query

> Claude Code 的 WebFetch 打不开大多数官方文档？
> Playwright 能打开，但 AI 不知道该怎么操作，只能瞎摸索？
>
> **Tech Doc Query** 把 15+ 站点的实战经验沉淀为一套标准化流程——诊断、提取、回退、汇报，
> 让 Claude Code 面对任何官方文档都像老手一样稳。

## 为什么有这个 Skill

Claude Code 内置的 WebFetch 对官方文档站点几乎束手无策——认证墙、iframe 嵌套、
JS 动态渲染、Cloudflare 拦截，随便一个就能让它哑火。

接入了 Playwright MCP 之后，浏览器能打开了，但新问题来了：**AI 不知道怎么操作。**
该用 browser_click 还是 browser_run_code_unsafe？iframe 里的内容怎么取？
ref 为什么总是过期？每次都要从零踩坑，效率极低。

于是有了这个 Skill——**把 15+ 个真实站点的诊断和提取经验固化为可执行的流程**，
让 AI 不再瞎摸索，每次查询都按同一套经过验证的规则来。

## 能做什么

| 场景 | 说明 |
|------|------|
| 查 API 参考 | "查一下 MATLAB conv 函数的官方说明" |
| 读官方教程 | "看 ROS 2 官方安装教程" |
| 总结文档概述 | "总结 Claude Code 的官方使用方式" |
| 跨多站点查询 | 同一个会话里先后查 ANSYS、Oracle、SAP |
| 需要登录的文档 | 自动停住等你手动登录，之后继续提取 |
| WebFetch 打不开的 | iframe、认证墙、JS 渲染——全走 Playwright 浏览器 |

## 覆盖的文档站点类型（15+ 已实测）

| 类型 | 代表站点 | 自动适配 |
|------|---------|---------|
| iframe 内容 | ANSYS Help | 使用 `browser_run_code_unsafe` + frame context |
| Docusaurus/MDX | DeepSeek API、Claude Code Docs、Auth0 | 标准 DOM 提取 |
| Sphinx/Read the Docs | ROS 2、toolbox | 过滤 RTD 注入元素 |
| GitBook SPA | GitBook Docs | SPA 等待渲染 |
| GitHub Docs | GitHub REST API | 大页面自动降级 heading 定位 |
| Oracle/SAP | 企业文档（iframe 用于 cookie） | 正确诊断 iframe≠内容 |
| Microsoft Learn | 微软文档 | Cookie 横幅处理 |
| VitePress | VitePress、Vue.js | body 仅 5KB，最干净 |
| Swagger/Redoc | API 浏览器 | 直接 fetch spec JSON |
| GNU/texinfo | GNU Bash 手册 | 单文件 524KB 自动切 heading 定位 |
| Twilio API | 开发者文档 | 双栏布局 |
| 认证页面 | DeepSeek 平台 | CAPTCHA + 验证码流程 |

## 环境要求

1. **Claude Code** （任意终端/IDE 版本）
2. **Playwright MCP** 已注册：在 Claude Code 中运行 `/mcp` 确认 `playwright` 在列表中
3. **浏览器 channel**：
   ```bash
   npx playwright install chrome
   ```

## 安装

```bash
# 克隆到 Claude Code skills 目录
mkdir -p ~/.claude/skills/tech-doc-query
cp SKILL.md ~/.claude/skills/tech-doc-query/SKILL.md
cp reference.md ~/.claude/skills/tech-doc-query/reference.md
```

安装后重启 Claude Code（或 `/clear`），skill 即生效。

## 使用方式

### 自动触发

只要你的请求包含以下关键词，skill 会自动激活：

**中文：** 官方文档、官方教程、官方使用方式、怎么使用、帮助文档、官方概述

**英文：** check the docs、official documentation、help page、verify syntax、API reference、getting started、usage guide、summarize docs

**其他触发条件：** 你贴了一个官方文档 URL、WebFetch 返回了 auth/domain 错误。

### 手动调用

```
/tech-doc-query
```

### 典型对话

```
你：https://docs.ros.org/en/jazzy/Installation.html 看 ROS 2 的官方安装方式

Claude：[启动浏览器 → 诊断 Sphinx/RTD 站点 → 提取内容]
        ROS 2 Jazzy 安装方式：
        - Ubuntu 24.04：推荐 deb packages
        - RHEL 9：推荐 RPM packages
        - Windows 10：Binary (VS 2019)
        ...
```

```
你：查一下 Oracle 官方文档里关于安装的内容

Claude：[诊断 Oracle Help Center：有 iframe 但不是内容容器 → article 提取]
        Oracle Database 26ai 安装文档目录：
        - Linux Installation Guides（5 篇）
        - Windows Installation Guides（2 篇）
        - ...
```

```
你：https://platform.deepseek.com/usage 查我的用量

Claude：[页面跳转到 /sign_in]
        这个页面需要登录。登录页已打开，你想怎么登录？
你：手机号 138xxxx，帮我点发送验证码
Claude：[填写手机号，点击发送]
        验证码已发送。人机验证需要你手动完成。
你：验证码是 123456
Claude：[填写验证码，登录成功 → 跳转到 /usage → 提取用量数据]
        你的用量：余额 ¥6.36，本月消费 ¥17.59...
```

## 工作原理

```
用户请求
  → Pre-Flight 诊断（Cloudflare？iframe？语义元素？站点类型？）
  → Method 1: browser_snapshot（100% 成功率，始终优先）
  → 内容太大？→ Method 3: heading-targeted 提取
  → 无语义元素？→ Method 2: DOM 容器回退
  → Swagger/Redoc？→ Method 4: fetch spec JSON
  → 超长页面？→ Method 5: section 边界分页
  → 结构化总结 + 原文 URL
```

## 限制

- **浏览器操作必须在主会话执行**——子 Agent 没有 Playwright MCP 访问权。多页面查询用 `browser_tabs` 串行切换。
- **Cloudflare 拦截的站点无法访问**——Stripe、OpenAI 等会被检测为 headless 浏览器。skill 会自动检测并告知用户。
- **需要登录的站点**——skill 会停下来等你手动完成 CAPTCHA/验证码。

## 文件结构

```
tech-doc-query/
├── SKILL.md        # 主 skill 文档（Pre-Flight + 6 步流程 + 5 种提取方法 + 规则）
└── reference.md    # 站点案例库（15+ 站点 + 失效模式表 + 故障排除）
```

## 贡献新的站点

遇到了 reference.md 没覆盖的文档站点？运行一次查询后，把诊断结果按模板加入 `reference.md` 即可：

```markdown
### 站点名 (域名)
- **Type:** server-rendered / SPA / iframe
- **iframe:** none / name=X
- **Semantic:** `<main>` ✅ / `<article>` ✅ / none
- **Extraction:** 最佳方法
- **Quirks:** 特殊注意事项
```

## 许可

MIT
