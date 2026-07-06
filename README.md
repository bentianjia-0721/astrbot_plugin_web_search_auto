# Web Search Auto - 自动网络搜索

参考 Claude Code 的 WebSearch / WebFetch 工具设计。

LLM 无需命令前缀，无需 function calling。后台自动搜索，结果注入上下文，LLM 优先使用搜索结果回复。

默认爬取 Bing（www.bing.com），零部署。

## 工作原理

```
用户消息 → 插件拦截 → 后台自动搜索 → 搜索结果注入 system prompt（高优先级）
        → LLM 基于搜索上下文自然回复（优先引用搜索结果）
```

不需要 LLM 支持 function calling，弱模型也能用。

## 特性

- **自动触发搜索**：无需手动命令，LLM 收到消息前自动搜索
- **强制优先级**：搜索结果注入 system prompt 顶部，确保 LLM 优先使用
- **零部署**：默认使用 Bing 搜索，无需 API Key
- **智能缓存**：15 分钟 TTL 缓存，避免重复请求
- **多后端支持**：Bing / DuckDuckGo / SearXNG 可选

## 安装

1. 插件目录放入 `AstrBot/data/plugins/astrbot_plugin_web_search_auto/`
2. WebUI 重载插件，默认即可用（Bing 搜索，无需任何配置）

## 搜索后端

| 后端 | 说明 |
|------|------|
| **bing**（默认） | 爬取 www.bing.com |
| duckduckgo | ddgs 库，需 `pip install ddgs` |
| searxng | 自托管 SearXNG API |

## 配置

| 配置项 | 默认值 | 说明 |
|--------|--------|------|
| `search_backend` | `bing` | bing / duckduckgo / searxng |
| `proxy` | `` | HTTP 代理，如 `http://127.0.0.1:7890` |
| `max_results` | `6` | 注入上下文的搜索结果数（增加数量提高准确性） |
| `cache_ttl` | `900` | 页面缓存秒数 (0=禁用) |
| `cache_max_size` | `100` | 缓存最大条目数 |
| `fetch_timeout` | `15` | 页面获取超时秒数 |
| `fetch_max_chars` | `10000` | 提取正文最大字符数 |

## 使用示例

```
用户: LuckPerms 怎么给玩家权限？
→ 插件后台搜 "LuckPerms 怎么给玩家权限"
→ 搜索结果（6条）注入 LLM 上下文顶部（高优先级）
→ LLM 优先基于搜索结果回答: LuckPerms 通过 /lp user <玩家> permission set <权限> 来设置...

用户: Claude Fable 5 什么时候发布的？
→ 自动搜索 → LLM 基于最新搜索结果回答发布日期和特性
```

## 优化说明（v2.1.0）

- **移除短消息过滤**：不再要求消息长度 ≥4，所有消息都触发搜索
- **增强上下文提示**：明确指示 LLM 优先使用搜索结果
- **增加默认结果数**：从 5 条增加到 6 条，提供更丰富的上下文
- **改进日志**：搜索失败时记录警告，便于排查
- **Bing 域名**：从 cn.bing.com 改为 www.bing.com（国际版）

## 依赖

```
aiohttp beautifulsoup4 lxml
# duckduckgo 后端需额外: ddgs
```
