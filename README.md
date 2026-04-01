# 百度地图 Agent Plan Skills

本仓库用于存放百度地图 Agent Plan 相关的 AI 助手 Skills，供支持 Skills 的客户端在对话中按需加载。当前包含一个 `baidu-ai-map` Skill，用于直连调用百度地图 Agent Plan 的地点检索、路线规划、地理编码、逆地理编码和天气查询能力。

## 包含的 Skill

| Skill | 说明 |
|-------|------|
| `baidu-ai-map` | 百度地图 Agent Plan 直连 Skill，覆盖 `place`、`direction`、`geocoding`、`reverse_geocoding`、`weather` 五类能力。调用时统一从环境变量 `BAIDU_MAP_AUTH_TOKEN` 读取鉴权凭证。 |

## 如何使用

### 1. 获取 Skill

方式 A：克隆仓库

```bash
git clone https://github.com/baidu-maps/map-skills.git
cd map-skills
```

方式 B：从 GitHub Release 下载

你也可以直接从 Releases 下载 `baidu-ai-map.zip`，解压后得到 `baidu-ai-map/` 文件夹：

```bash
unzip baidu-ai-map.zip
```

### 2. 配置鉴权

调用 Skill 前，需要先设置百度地图 Agent Plan 的 SK：

```bash
export BAIDU_MAP_AUTH_TOKEN="你的SK"
```

如果还没有 SK，可前往 <https://lbs.baidu.com/apiconsole/agentplan> 申请。

### 3. 将 Skill 注册到你的 AI 助手

如果你是从仓库克隆使用，请把 `skills/baidu-ai-map` 链接或复制到当前环境对应的 skills 目录。

如果你是从 Release 下载使用，请把解压后的 `baidu-ai-map` 目录链接或复制到当前环境对应的 skills 目录。

**OpenClaw**

- 以下示例默认使用：`~/.openclaw/workspace/skills/`
- 如果你的 OpenClaw 环境使用的是 `~/.openclaw/skills/`，请将下面命令中的目标路径整体替换为对应目录
- 本 Skill 在 `SKILL.md` 中声明了 OpenClaw 依赖：
  - `curl`
  - `BAIDU_MAP_AUTH_TOKEN`
- 注册前可先确保目录存在：

```bash
mkdir -p ~/.openclaw/workspace/skills
```

- 如果使用仓库目录注册（推荐）：

```bash
ln -sfn "$(pwd)/skills/baidu-ai-map" ~/.openclaw/workspace/skills/baidu-ai-map
```

- 如果使用 Release 解压目录注册：

```bash
ln -sfn "$(pwd)/baidu-ai-map" ~/.openclaw/workspace/skills/baidu-ai-map
```

- 如果你的 OpenClaw 环境使用的是 `~/.openclaw/skills/`，可参考：

```bash
ln -sfn "$(pwd)/skills/baidu-ai-map" ~/.openclaw/skills/baidu-ai-map
```

### 4. 在对话中使用

当你的问题涉及「百度地图」「Agent Plan」「地点检索」「路线规划」「地理编码」「逆地理编码」「天气查询」等场景时，客户端会加载 `baidu-ai-map` 的说明，帮助助手按该 Skill 中定义的参数约束和调用方式输出结果或代码。

## 许可

`baidu-ai-map` 在 `SKILL.md` frontmatter 中声明的许可证为 MIT。
