# Windows 上安装腾讯 SkillHub，并接入 Tavily Web Search 的完整实践

这份文档面向国内用户，目标不是泛泛介绍，而是把一条真正能跑通的 Windows 路径讲清楚：

1. 安装腾讯 SkillHub
2. 用 `skillhub` 安装 `Tavily Web Search`
3. 配置 `TAVILY_API_KEY`
4. 让 OpenClaw 正确识别并调用这个 skill
5. 解决“聊天里不会自动执行 SkillHub 安装命令”的问题

对国内用户来说，SkillHub 的价值不只是一个技能列表页，而是它把“更顺畅的下载体验、精选技能入口、以及更大规模的 Skills 生态”连接在了一起。真正有用的不是“看见技能”，而是“能在自己的机器上顺利用起来”。

## 一、适用场景

这套方案适合：

- Windows 用户
- 正在使用 OpenClaw
- 想让 AI 获得更强的搜索能力
- 希望通过 SkillHub 管理和安装 skills
- 不想在安装、权限、路径、编码问题上反复试错

## 二、环境说明

本文基于以下环境验证：

- Windows
- OpenClaw `2026.3.8`
- 腾讯 SkillHub
- Tavily Web Search skill

## 三、安装腾讯 SkillHub

官网入口：

[https://skillhub.tencent.com/#featured](https://skillhub.tencent.com/#featured)

安装完成后，先确认命令是否可用：

```powershell
skillhub --help
```

如果这条命令能够输出帮助信息，说明 SkillHub CLI 已经安装成功。

## 四、搜索并安装 Tavily Web Search

### 1. 搜索

```powershell
skillhub search tavily-search
```

实测可搜索到：

- `tavily-search`
- 名称：Tavily Web Search
- 版本：`1.0.0`

### 2. 安装

```powershell
skillhub install tavily-search
```

成功输出示例：

```text
Downloading: http://lightmake.site/api/v1/download?slug=tavily-search
Installed: tavily-search -> D:\fen\openclaw\skills\tavily-search
```

这说明 SkillHub 已经成功把 skill 安装到本地目录。

## 五、确认 skill 结构

安装后的目录结构类似：

```text
skills/tavily-search/
├─ SKILL.md
├─ _meta.json
└─ scripts/
   ├─ search.mjs
   └─ extract.mjs
```

这个 skill 的关键信息包括：

- 运行依赖：`node`
- 必需环境变量：`TAVILY_API_KEY`

## 六、配置 `TAVILY_API_KEY`

Tavily skill 需要 `TAVILY_API_KEY`。

可以采用两种方式：

### 方式 A：系统环境变量

```powershell
setx TAVILY_API_KEY "你的Key"
```

### 方式 B：OpenClaw skill 配置注入

```json
{
  "skills": {
    "entries": {
      "tavily": {
        "enabled": true,
        "env": {
          "TAVILY_API_KEY": "REDACTED"
        }
      }
    }
  }
}
```

如果你使用本地配置文件管理 Key，也可以先读本地文件，再由 OpenClaw 注入到 skill 环境中。

## 七、让 OpenClaw 加载项目目录下安装的 skill

如果 `skillhub install` 把 skill 装到了项目目录，而不是 `~/.openclaw/skills`，需要把目录显式加入 `skills.load.extraDirs`：

```json
{
  "skills": {
    "load": {
      "extraDirs": [
        "D:\\fen\\openclaw\\skills"
      ]
    }
  }
}
```

这样 OpenClaw 才能在主实例中识别到这些通过 SkillHub 安装到项目目录的 skills。

## 八、最容易踩坑的一步：不是 SkillHub 不能装，而是 OpenClaw 没有 exec

很多人会遇到这个现象：

- `skillhub install tavily-search` 手动执行是成功的
- 但是在聊天里让 OpenClaw 安装 skill 时，它却说没有命令执行能力

这里的关键是：

- SkillHub 插件只是在提示词里告诉 agent：优先执行 `skillhub search/install`
- 它本身不会自动给 agent 提供本地命令执行能力

所以真正要解决的是 OpenClaw 的工具权限配置。

## 九、推荐的 OpenClaw 配置

推荐保留聊天场景为主的 `messaging` 配置，同时放开受控的命令执行能力：

```json
{
  "tools": {
    "profile": "messaging",
    "allow": [
      "group:runtime",
      "group:fs"
    ],
    "exec": {
      "host": "gateway",
      "security": "allowlist",
      "ask": "on-miss"
    }
  }
}
```

这套配置的意义：

- `profile: messaging` 保留消息型 agent 的基础能力
- `group:runtime` 允许 `exec`
- `group:fs` 允许安装流程常见的文件读写能力
- `tools.exec` 让命令执行不是完全裸奔，而是走受控策略

## 十、重启并验证

重启 gateway：

```powershell
openclaw gateway restart
```

检查 gateway：

```powershell
openclaw gateway status
```

检查 skill：

```powershell
openclaw skills list
openclaw skills info tavily
```

理想结果：

- `tavily` 显示 `ready`
- 来源正常
- 路径正常
- OpenClaw 已能识别该 skill

## 十一、已知问题

### 1. Windows 控制台编码问题

在部分 Windows 环境中，`skillhub search tavily-search` 可能因为控制台编码导致 `UnicodeEncodeError`。

这通常不代表 search 本身失败，而是输出中文描述时触发了编码异常。

### 2. 聊天里“不会自动安装 skill”

这通常不是 SkillHub 本身的问题，而是当前 OpenClaw 会话没有 `exec`。

### 3. 敏感信息泄露风险

公开分享时必须统一脱敏：

- API Key
- GitHub Token
- Bot Token
- 账号 ID
- 配置文件中的敏感字段

## 十二、结论

这套链路在 Windows 上是能跑通的。

真正需要解决的不是“SkillHub 能不能装 Tavily”，而是：

1. Windows 下的安装与编码差异
2. OpenClaw 是否加载了项目技能目录
3. OpenClaw 当前聊天会话是否拥有受控的命令执行能力

只要把这三层打通，SkillHub + OpenClaw + Tavily Web Search 就可以成为一套对国内用户非常实用的 AI Skills 增强方案。

