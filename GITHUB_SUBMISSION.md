# Bilibili 综合热榜爬取项目说明（GitHub 提交版）

## 1. 项目简介
本项目用于抓取 Bilibili 综合热榜页面：

- 页面地址：`https://www.bilibili.com/v/popular/all`

项目支持两种接入方式：

1. Bright Data + Haystack（主通道）
2. 本地直连抓取（回退通道）

输出统一为 JSON，字段包含视频标题、作者、播放量、点赞量、弹幕量、视频链接等，便于后续做数据分析、榜单追踪和可视化。

---

## 2. 技术方案
### 2.1 核心流程
1. 先校验目标页面可访问（`/v/popular/all`）。
2. 调用该页面对应的数据接口：
   - `https://api.bilibili.com/x/web-interface/popular`
3. 解析并标准化字段。
4. 写入本地 JSON 文件。

### 2.2 通道策略
通过环境变量 `SCRAPE_CHANNEL` 控制：

- `local`：只走本地通道（真实抓取）
- `brightdata`：只走 Bright Data 通道
- `auto`：优先 Bright Data，失败自动回退本地

---

## 3. 项目结构
```text
bili_hot_local_fetcher.py      # 本地真实抓取实现
bili_hot_custom_tool.py        # Bright Data SDK 风格 + 回退
bili_hot_mcp.py                # Bright Data MCP + 回退
requirements.txt
.env.example
output/
```

---

## 4. 核心代码（节选）
### 4.1 目标页面与数据接口定义
```python
BILI_POPULAR_ALL_PAGE_URL = "https://www.bilibili.com/v/popular/all"
BILI_POPULAR_ALL_API_URL = "https://api.bilibili.com/x/web-interface/popular"
```

### 4.2 抓取热门榜单数据（本地通道）
```python
response = session.get(
    BILI_POPULAR_ALL_API_URL,
    params={"page": page, "page_size": page_size},
    timeout=timeout,
)
response.raise_for_status()
payload = response.json()

if payload.get("code") != 0:
    raise RuntimeError(f"Bilibili popular/all API returned error: {payload}")

videos = (payload.get("data") or {}).get("list") or []
```

### 4.3 Bright Data SDK 主通道 + 自动回退
```python
if channel in {"auto", "brightdata"}:
    try:
        records = _run_brightdata_custom_tool(...)
    except Exception as exc:
        if channel == "brightdata":
            raise
        records = _run_local_channel(...)
```

### 4.4 统一输出 JSON
```python
out_path = Path("output/bili_hot_custom_tool.json")
out_path.parent.mkdir(parents=True, exist_ok=True)
out_path.write_text(
    json.dumps(records, ensure_ascii=False, indent=2),
    encoding="utf-8",
)
```

---

## 5. 输出字段说明
每条视频记录示例字段：

- `rank`：榜单序号
- `title`：视频标题
- `author`：UP 主
- `play`：播放量
- `like`：点赞量
- `danmaku`：弹幕量
- `duration`：时长
- `pubdate`：发布时间
- `bvid` / `aid`：视频唯一标识
- `url`：视频地址
- `desc`：简介
- `source_page`：来源页面（`/v/popular/all`）
- `source_api`：来源接口

---

## 6. 运行方式
### 6.1 安装依赖
```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
pip install -r requirements.txt
Copy-Item .env.example .env
```

### 6.2 执行脚本
```powershell
python .\bili_hot_custom_tool.py
python .\bili_hot_mcp.py
```

输出文件：

- `output/bili_hot_custom_tool.json`
- `output/bili_hot_mcp.json`

---

## 7. 说明与边界
1. 项目不使用写死 mock 榜单数据，输出为实时抓取结果。
2. 已包含 Bright Data 通道，但在额度不足/权限不足时会自动回退到本地通道（`auto` 模式）。
3. 建议控制抓取频率，遵守目标站点条款与当地法律法规。

---

## 8. 可继续扩展
1. 增加“按时间周期（日/周）对比热门变化”功能。
2. 增加“分类榜（动画、游戏、影视）”抓取策略。
3. 增加可视化（如 Streamlit/前端图表）用于展示榜单趋势。
