# Windows 装腾讯 SkillHub，再接 Tavily Search，这套我已经跑通了

如果你也是 Windows 用户，最近又想把 OpenClaw 的技能生态用起来，那你大概率会遇到这几个问题：

- SkillHub 到底怎么装
- `tavily-search` 能不能装
- 为什么 AI 不会自动帮我安装
- `TAVILY_API_KEY` 放哪儿

我这次把整条链路实测跑通了，而且结论很直接：

不是 SkillHub 不能装。  
真正卡人的，很多时候是 OpenClaw 当前会话没有命令执行能力。

## 先说结果

我最终跑通的是：

1. Windows 安装腾讯 SkillHub
2. `skillhub install tavily-search`
3. 配 `TAVILY_API_KEY`
4. 让 OpenClaw 识别 `tavily`
5. 让聊天里的 agent 以后能直接执行 `skillhub install ...`

最终状态：

- `tavily` 在 OpenClaw 里显示 `ready`
- gateway 正常
- SkillHub 能正常给 Windows 环境装 skill

## 第一步：装腾讯 SkillHub

入口：

[https://skillhub.tencent.com/#featured](https://skillhub.tencent.com/#featured)

如果你是 Windows 用户，我更建议直接用整理好的适配版，不要临时自己拼一套。  
原因很简单：Windows 下最烦的不是会不会装，而是每一步都多一个小坑。

装完先检查：

```powershell
skillhub --help
```

## 第二步：搜 Tavily Search

```powershell
skillhub search tavily-search
```

我这边能搜到：

- `tavily-search`
- Tavily Web Search
- 版本 `1.0.0`

注意一点：

Windows 下 `skillhub search` 有概率因为控制台编码问题，在后半段报错。  
但这不等于 skill 不存在，很多时候结果已经搜出来了。

## 第三步：直接装

```powershell
skillhub install tavily-search
```

我这边实测安装成功，输出类似：

```text
Installed: tavily-search -> D:\fen\openclaw\skills\tavily-search
```

所以结论非常明确：

`tavily-search` 能装，SkillHub 本身没问题。

## 第四步：配 Key

这个 skill 需要：

```text
TAVILY_API_KEY
```

你可以放环境变量里，也可以像我这次一样放本地配置文件里，再由 OpenClaw 注入。

重点只有一个：

所有 Key 一定要脱敏，不要出现在截图和公开文档里。

## 第五步：为什么 AI 之前不帮你装

这才是核心坑。

很多人看到这句：

> 当前会话没有可用的命令执行工具

就会以为是 SkillHub 不行。

其实不是。

真实原因是：

- SkillHub 插件只是告诉 OpenClaw：优先执行 `skillhub search/install`
- 但当前会话如果没有 `exec`
- AI 就知道该怎么做，但没法真的执行

## 正确改法

把 OpenClaw 工具配置改成：

```json
{
  "tools": {
    "profile": "messaging",
    "allow": ["group:runtime", "group:fs"],
    "exec": {
      "host": "gateway",
      "security": "allowlist",
      "ask": "on-miss"
    }
  }
}
```

这套配置的好处：

- 能让 agent 真正执行 `skillhub install ...`
- 但不是无脑放开
- 白名单外命令还会审批

## 第六步：验证是不是 ready

重启：

```powershell
openclaw gateway restart
```

再查：

```powershell
openclaw skills list
openclaw skills info tavily
```

我这边最终结果是：

- `tavily` = `ready`
- OpenClaw 已识别
- Key 已注入

## 一句话总结

这套链路能跑通。  
问题不在 SkillHub，问题主要在：

- Windows 编码
- OpenClaw 的 exec 权限
- Key 怎么配置

如果你也是 Windows 用户，其实最值钱的不是“命令会不会写”，而是“有没有人帮你把坑提前踩完”。

如果你需要我打包好的适合 Windows 的 SkillHub，记得关注、点赞、留言。  
我后面会继续把这套方案整理成更适合普通用户直接上手的版本。

