---
name: arcgis-agent
description: 'ArcGIS Agent - use natural language to drive ArcGIS Pro via the arcpy-mcp-server HTTP API (1300+ spatial analysis tools: buffer/clip/intersect, slope/aspect/viewshed, kriging/IDW, hotspot/Morans I, network analysis, geocoding and more). Use when the user asks for GIS/spatial analysis, ArcGIS automation, or arcpy operations. Requires Windows + ArcGIS Pro 3.x. 通过自然语言驱动 ArcGIS Pro 的 1300+ arcpy 空间分析工具。'
version: 1.0.0
metadata:
  author: based on zhaojj662/arcpy-mcp-server (MIT)
  source: https://github.com/zhaojj662/arcpy-mcp-server
  tags: [gis, arcgis, arcpy, spatial-analysis, mcp]
---

# ArcGIS Agent

驱动本机 ArcGIS Pro 的 1300+ arcpy 工具，让 AI 直接用自然语言完成空间分析。

架构：`本技能(Agent) --HTTP--> scripts/server.py (arcpy) --> ArcGIS Pro 3.x`

## 前置条件

- Windows 10/11 + **ArcGIS Pro 3.x**（有效许可证）——arcpy 只能在 ArcGIS Pro 自带的 Python 中运行
- 服务端 HTTP 地址：`http://127.0.0.1:8765`（仅本机监听）

## 第一步：确保服务在运行

先探测服务是否已启动：

```bash
curl -s http://127.0.0.1:8765/health
# {"status":"ok","server":"arcpy-mcp-server","version":"2.0","tools":1300}
```

若未启动，后台启动服务（用 ArcGIS Pro 自带 Python）：

```bash
"C:/Program Files/ArcGIS/Pro/bin/Python/envs/arcgispro-py3/python.exe" "<本技能目录>/scripts/server.py" 8765
```

Windows 下也可运行 `scripts/start_server.bat`（前台窗口，Ctrl+C 停止）。

## 第二步：调用工具（纯 HTTP，无需 MCP 客户端）

Agent 直接用 curl 调用即可：

```bash
# 列出全部模块及工具数量
curl -s http://127.0.0.1:8765/modules

# 列出某个模块的全部工具名（如分析模块）
curl -s http://127.0.0.1:8765/module/analysis

# 执行工具：POST /call  body = {"name": "模块_工具名", "arguments": {...}}
curl -s -X POST http://127.0.0.1:8765/call -H "Content-Type: application/json" \
  -d '{"name":"analysis_Buffer","arguments":{"in_features":"C:/data/roads.shp","out_feature_class":"C:/out/roads_buf.shp","buffer_distance_or_field":"500 meters"}}'
```

工具命名规则：`{模块名}_{工具名}`，如 `analysis_Buffer`、`sa_Reclassify`、`ga_Kriging`。
不确定工具名时，先查 `/modules` → `/module/{name}`，再用 `/tool/{工具id}` 确认。
参数为 arcpy 该工具的关键字参数（详见 ArcGIS Pro 官方文档中的对应工具签名）。

## 模块速查

| 模块 | 工具数 | 用途 |
|---|---|---|
| management | 392 | 数据管理、投影、字段、拓扑 |
| sa (Spatial Analyst) | 355 | 坡度坡向、重分类、可视域、水文 |
| ddd (3D Analyst) | 144 | DEM、等高线、天际线、点云 |
| nax (Network Analyst) | 61 | 路径、服务区、OD 矩阵 |
| stats (空间统计) | 44 | Moran's I、Gi*、KDE、GWR |
| ga (地统计) | 39 | 克里金、IDW、EBK |
| analysis | 38 | 缓冲区、裁剪、相交、擦除、泰森多边形 |
| conversion / na / cartography / geocoding / stpm / edit / server / sharing | 其余 | 格式转换、制图、地址匹配、时空立方体等 |

## 典型调用链（复合分析）

选址类需求按"缓冲区→叠加→筛选"链条自动拆解为多次 /call，例：
医院选址（距主干道 500m 内、坡度 < 5°）：
1. `analysis_Buffer` 道路 500m → 2. `ddd_Slope` DEM 求坡度 → 3. `sa_Reclassify` 坡度二值化 → 4. `analysis_Intersect` 叠加取交集

更多示例（缓冲区、克里金、热点分析、可视域、投影、字段计算等）见 `references/examples.md`。

## 错误处理

返回 `{"status":"error","message":"...","suggestion":"..."}`，常见代码：
- `000732`：数据集不存在 → 先确认路径/扩展名
- `000725`：输出已存在（服务端已自动 overwriteOutput=True，一般不出现）
- `000735` / `000622`：参数格式或类型错误 → 核对工具签名

结果中 `info` 含执行耗时；若输出为矢量/栅格数据，还会带要素数、几何类型或栅格尺寸。

## 安全与配置

- 服务只监听 `127.0.0.1`，不会暴露到局域网
- 路径白名单：默认只允许 `C:/GIS-AI-Course/` 下的 .shp/.tif/.gdb 读写；用环境变量 `ARCPY_ALLOWED_PATHS`（分号分隔多个路径）放开其他目录，例如
  `set ARCPY_ALLOWED_PATHS=C:/GIS-AI-Course/;D:/data/`
- 模块白名单：编辑 `scripts/server.py` 中 `INCLUDE_MODULES` 只暴露需要的模块
- 谨慎执行 `management`/`edit` 模块的写操作前，向用户确认目标数据
