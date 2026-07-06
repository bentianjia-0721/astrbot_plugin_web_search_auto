# Web Search Auto - 自动网络搜索

参考 Claude Code 的 WebSearch / WebFetch 工具设计。

LLM 无需命令前缀，无需 function calling。后台自动搜索，结果注入上下文，LLM 优先使用搜索结果回复。

默认爬取 Bing（www.bing.com），零部署。

## 技术架构

### 核心技术栈

- **异步 HTTP**：基于 `aiohttp` 的全异步网络请求，支持高并发搜索和页面抓取
- **HTML 解析**：`BeautifulSoup4` + `lxml` 解析器，提取搜索结果和页面正文
- **缓存系统**：FIFO + TTL 双重策略，内存缓存页面内容（15分钟过期）
- **事件拦截**：AstrBot `@filter.on_llm_request()` 钩子，在 LLM 请求前注入搜索上下文
- **多后端适配**：工厂模式设计，支持 Bing / DuckDuckGo / SearXNG 灵活切换

### 工作原理

```
用户消息 → on_llm_request 拦截 → 提取查询文本 → 去重检查
                                              ↓
后端搜索（并发） → HTML 解析 → 域名过滤 → 结果排序
                                              ↓
构建上下文 Markdown → 注入 system prompt 顶部 → LLM 处理
                                              ↓
LLM 基于搜索结果生成回复 → 自然引用来源 → 返回用户
```

### 关键技术特性

1. **零 Function Calling 依赖**
   - 不依赖 LLM 的 tool use 能力，弱模型也能使用
   - 搜索结果作为上下文注入 system prompt，LLM 自然处理

2. **智能搜索后端**
   - **Bing Scraper**：直接爬取 `www.bing.com` 搜索结果，无需 API Key
   - **DuckDuckGo**：使用 `ddgs` 库，支持多引擎聚合（Brave/Yahoo/Bing）
   - **SearXNG**：支持自托管的 SearXNG 实例（JSON API）

3. **高性能缓存**
   - **FIFO 淘汰**：达到容量上限时，删除最早的缓存项
   - **TTL 过期**：默认 15 分钟（900秒）自动失效，避免过期数据
   - **SHA256 Key**：URL 哈希作为缓存键，避免冲突

4. **上下文优化**
   - **优先级标记**：`[PRIORITY CONTEXT]` 提示 LLM 优先使用搜索结果
   - **结构化注入**：Markdown 格式排版，包含标题、URL、摘要
   - **摘要截断**：每条结果最多 300 字符，平衡信息量与 token 消耗

5. **容错与日志**
   - 搜索失败时记录 warning 日志，不影响 LLM 正常回复
   - 网络超时、HTTP 错误自动捕获，返回友好错误信息

### 后端实现细节

#### Bing Scraper
```python
# 直接 HTTP GET www.bing.com/search
# CSS 选择器提取: li.b_algo > h2 > a
# 无需认证，国内外均可访问
```

#### DuckDuckGo
```python
# 使用 ddgs 库的 DDGS.text() 方法
# 支持代理配置，同步调用转异步（run_in_executor）
```

#### SearXNG
```python
# JSON API: GET {base_url}/search?format=json&q={query}
# 需要自托管 SearXNG 服务
```

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

| 后端 | 说明 | 依赖 |
|------|------|------|
| **bing**（默认） | 爬取 www.bing.com | 无 |
| duckduckgo | ddgs 库 | `pip install ddgs` |
| searxng | 自托管 SearXNG API | 需部署 SearXNG 服务 |

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

## 性能指标

- **搜索延迟**：通常 200-500ms（Bing 后端）
- **缓存命中率**：相同查询 15 分钟内 100% 命中
- **并发支持**：基于 asyncio，理论无上限
- **内存占用**：~10KB/条缓存（100 条约 1MB）

## 依赖

```
aiohttp beautifulsoup4 lxml
# duckduckgo 后端需额外: ddgs
```
