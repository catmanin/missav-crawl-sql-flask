# MissAV Exhaustive Crawler & m3u8 Link Database Project

## 1. 项目概述 (Project Overview) - v2.0

本项目是一个大规模数据采集与管理系统，旨在实现一个明确且具有挑战性的目标：

通过**番号穷举法**，系统性地扫描 MissAV 网站多达 **40万个** 可能的页面，主要任务是**抓取并提取有效的 m3u8 视频流链接**。整个抓取过程将在 **Google Colab** 环境中执行，以利用其免费的计算资源。

采集到的数据将被持久化到一个 **SQLite 数据库**中，并提供多种导出格式，包括可直接播放的 **DPL 播放列表**、可浏览的 **HTML 报告页**，并为最终通过 **Flask** 搭建Web应用提供数据基础。

## 2. 核心工作流 (Core Workflow)

项目将遵循以下核心流程：

1.  **番号生成 (Code Generation)**:
    * 实现一个智能生成器，按 `a-1` 到 `a-10000`, `b-1`... `z-10000`, `aa-1`... `zz-10000` 的模式穷举生成番号。
    * 生成器需要可控，能够生成指定数量（例如40万）的番号。

2.  **并发抓取 (Concurrent Crawling)**:
    * 在 Google Colab Notebook 中执行抓取任务。
    * 利用多线程 (`concurrent.futures.ThreadPoolExecutor`) 来并发发送 HTTP 请求，最大化利用网络资源，缩短总耗时。
    * **关键策略**: 实现**断点续传/检查点 (Checkpointing)** 机制，定期将进度保存到 Google Drive，防止 Colab 会话中断导致数据丢失。

3.  **数据解析与提取 (Parsing & Extraction)**:
    * 对返回的 HTML 页面进行解析。
    * 核心目标是定位并提取 `m3u8` 链接。这需要深入分析页面 `<script>` 标签内的 Javascript 代码，可能需要使用正则表达式进行精确匹配。
    * 同时记录页面的状态（成功找到、页面不存在404、请求错误等）。

4.  **数据持久化 (Data Persistence)**:
    * 使用 Colab 环境内建的 `sqlite3` 库。
    * 将数据库文件 (`.db`) 直接存储在挂载的 Google Drive 上，确保数据安全和持久。
    * 每次抓取成功或失败后，都将结果（番号、m3u8链接、状态）实时写入数据库。

5.  **结果导出 (Result Exporting)**:
    * 编写独立的脚本函数，从 SQLite 数据库中读取数据。
    * 生成 `playlist.dpl` 文件，每行一个有效的 `m3u8` 链接。
    * 生成 `report.html` 文件，用表格形式展示所有成功抓取的番号及其对应的 `m3u8` 链接。

6.  **Web 应用展示 (Flask UI)**:
    * （可选最终阶段）开发一个简单的 Flask 应用，连接到 Google Drive 上的 SQLite 数据库，提供一个带搜索、分页功能的网页来浏览所有抓取到的数据。

## 3. Google Colab 执行计划

这是一个长达24小时以上的任务，必须为 Colab 的环境特性进行专门设计。

1.  **环境设置**:
    * **挂载 Drive**: `from google.colab import drive; drive.mount('/content/drive')` 必须是第一步。
    * **安装依赖**: `!pip install requests beautifulsoup4 tqdm`。

2.  **进度管理 (Checkpointing)**:
    * **数据库即进度**: 每次都向数据库 `INSERT OR REPLACE` 数据，这本身就是一种进度记录。
    * **启动时检查**: 爬虫启动时，首先从数据库中查询最后一个成功抓取的番号，或者查询所有已尝试过的番号，然后从下一个未尝试的番号开始，实现断点续传。
    * **状态日志**: 可以在 Google Drive 上创建一个简单的 `status.log` 文件，每处理完1000个番号就记录一下当前时间和最后一个番号，方便人工跟踪。

3.  **并发控制**:
    * Google Colab 的资源并非无限。并发线程数建议设置在 `10` 到 `30` 之间，需要进行测试以找到最佳平衡点，避免因请求过于密集而被目标服务器临时封禁。

## 4. 数据库设计 (Schema for SQLite)

数据库结构应尽可能精简，专注于核心任务。

**`videos` 表:**

| 字段名 (Column) | 数据类型 (Type) | 说明 (Notes) |
| :--- | :--- | :--- |
| `code` | TEXT | 主键。通过穷举法生成的番号，如 `abc-123`。 |
| `m3u8_link` | TEXT | 核心资源！提取到的完整 `m3u8` 链接。若未找到则为 `NULL`。 |
| `status` | TEXT | 抓取状态。建议值为: `SUCCESS`, `NOT_FOUND`, `REQUEST_ERROR`, `PARSE_ERROR`。 |
| `checked_at` | TIMESTAMP | 本次检查的时间戳。方便后续重新检查失败的条目。 |

**示例 SQL 创建语句:**
```sql
CREATE TABLE IF NOT EXISTS videos (
    code TEXT PRIMARY KEY,
    m3u8_link TEXT,
    status TEXT NOT NULL,
    checked_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

## 5. 输出文件格式定义

### `playlist.dpl` (PotPlayer Playlist)
纯文本文件，格式极其简单。每行一个有效的 m3u8 链接。
```
# DPL playlist generated on 2025-07-13

[https://domain.com/path/to/video1.m3u8](https://domain.com/path/to/video1.m3u8)
[https://domain.com/path/to/video2.m3u8](https://domain.com/path/to/video2.m3u8)
...
```

### `report.html` (HTML Report)
一个独立的 HTML 文件，包含一个可排序的表格。
```html
<!DOCTYPE html>
<html>
<head>
    <title>MissAV Crawl Report</title>
    </head>
<body>
    <h1>Crawl Report (400,000 attempts)</h1>
    <table>
        <thead>
            <tr>
                <th>Code</th>
                <th>M3U8 Link</th>
            </tr>
        </thead>
        <tbody>
            <tr>
                <td>abc-123</td>
                <td><a href="...">...</a></td>
            </tr>
            <tr>
                <td>def-456</td>
                <td><a href="...">...</a></td>
            </tr>
        </tbody>
    </table>
</body>
</html>
```

## 6. 代码实现蓝图 (Code Blueprint)

在 Colab Notebook 中，可以将代码划分为以下几个逻辑函数：

```python
# --- 1. 配置与初始化 ---
# GDrive 路径, DB 文件路径, 并发数等

# --- 2. 番号生成器 ---
def generate_codes(start_point=None, limit=400000):
    # 实现穷举逻辑
    # 如果提供了 start_point, 则从该点继续生成
    pass

# --- 3. 数据库交互 ---
def initialize_db():
    # 连接数据库，创建表
    pass

def get_crawled_codes():
    # 从数据库中获取所有已经尝试过的番号
    # 返回一个 set, 用于跳过已处理的番号
    pass

def save_result_to_db(code, m3u8_link, status):
    # 将单次抓取结果存入数据库
    pass

# --- 4. 核心抓取与解析 ---
def fetch_single_page(code):
    # 发送请求，处理异常
    # 解析 HTML, 重点用 re 或 bs4 提取 m3u8
    # 返回结果元组: (code, m3u8_link, status)
    pass

# --- 5. 主执行器 ---
def run_crawler():
    # 初始化
    # 获取已爬取的番号 set
    # 创建番号生成器
    # 使用 ThreadPoolExecutor 进行并发抓取
    # 使用 tqdm 显示进度条
    # 在循环中调用 fetch_single_page 和 save_result_to_db
    pass

# --- 6. 导出工具 ---
def export_to_dpl():
    # 查询数据库中所有成功的 m3u8_link
    # 写入 dpl 文件
    pass

def export_to_html():
    # 查询数据库...
    # 生成 HTML 文件
    pass

# --- 7. 主程序入口 ---
# if __name__ == "__main__":
#     run_crawler()
#     # 爬取结束后自动导出
#     export_to_dpl()
#     export_to_html()

```

## 7. 免责声明 (Disclaimer)

**本项目仅用于个人技术学习、数据处理和编程实践研究。**
- **请勿**将本项目及抓取到的任何数据用于商业或非法用途。
- 用户必须自行承担因使用本项目可能引发的所有风险和责任，包括但不限于IP被封禁、法律风险等。
- 请在目标网站可接受的范围内进行合理频率的抓取。
